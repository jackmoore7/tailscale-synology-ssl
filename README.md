# How to use Tailscale with a custom domain via Cloudflare and reverse proxy on a Synology NAS

## Introduction
I recently decided enough is enough, I want to access my self-hosted services while I'm on the go. A lot of people's first thought to accomplish this is **Tailscale**, a service where you can create a private network "bubble" of your chosen devices (called a Tailnet), which can be accessed via an oauth of your choosing. Upon signing up, you're given a unique subdomain, such as `tailb12345.ts.net`, which is used for DNS. As such, you can access your devices using this name instead of an IP address.

That's great and all, but I had wondered if I could take this a step further and use my own domain to access my devices. So, here we are. "But Jack" I hear you say, "Won't it be secure anyway if we're connecting through Tailscale?". And to that I say, well yes! But this method does come with a couple of benefits:
 - Not having to remember which port is which since it'll all be handled via reverse proxy
 - Not having to remember (frankly ugly) domain names

## Prerequisites
 - A Synology NAS
 - Docker installed on your NAS
 - A domain managed via Cloudflare
 - The ability to SSH into your NAS

## Initial setup
First off, you'll need to install Tailscale on your Synology NAS. There's a great video by Tailscale's YouTube channel that explains step-by-step how to install Tailscale, configure auto-update, and enable HTTPS: https://www.youtube.com/watch?v=0o2EhK-QvmY

Once you think you've got it set up, give it a test by going to `https://[nas name].[tailnet name].ts.net`. In my case, my nas is called `ds920`, and my tailnet name is `tailb12345`.
![brave_waKsC06BDN](https://github.com/user-attachments/assets/0099ef25-376f-4672-aec1-59129f2593f2)
And, we have a successful connection to our machine over HTTPS.
<img width="1470" alt="Screenshot 2024-12-19 at 15 05 14" src="https://github.com/user-attachments/assets/2be220e0-bbc4-43ff-a2cb-614a6cc14835" />

## Custom domain with wildcard
Now comes the fun part! We're going to use **Certbot** to get us a wildcard certificate by using the Cloudflare API to complete the DNS-01 challenge.

### Generating a Cloudflare API token

1. Create a User API token. 

Go to My Profile > API Tokens > Create Token. Give it whatever name you desire, and set parameters like so:
- Permissions:
  	- Zone: Zone: Read
	- Zone: DNS: Edit
- Zone Resources:
	- Include: Specific zone: `domain.com` (where `domain.com` is your domain)

Your summary should look like this: 
<img width="716" alt="Screenshot 2024-12-27 at 10 55 09" src="https://github.com/user-attachments/assets/78df83af-ce08-4c51-822c-45a606c55a16" />

2. Save the API key somewhere, because you won't be able to see it again.

### Running Certbot

This section will assume your volume is called `volume1` and that you have a folder called `docker` inside. Please make necessary adjustments if this doesn't align with your setup.

1. SSH into your NAS (if you don't know how to do this on your platform, there are a lot of guides out there)
2. Create the necessary directories for Certbot: `sudo mkdir -p /volume1/docker/certbot/{logs,lib_letsencrypt,etc_letsencrypt}`
3. Create the Certbot configuration file: `sudo touch /volume1/docker/certbot/lib_letsencrypt/cloudflare.ini`
4. Add your Cloudflare API token:
```
echo "dns_cloudflare_api_token = your-cloudflare-api-token" | \
sudo tee /volume1/docker/certbot/lib_letsencrypt/cloudflare.ini > /dev/null
```
5. Use the folling command to launch a temporary Docker container which runs Certbot and adds the necessary DNS records to your domain. Make sure you replace the domains and email with your own.
```
sudo docker run --rm \
  -v /volume1/docker/certbot/etc_letsencrypt:/etc/letsencrypt \
  -v /volume1/docker/certbot/lib_letsencrypt:/var/lib/letsencrypt \
  -v /volume1/docker/certbot/logs:/var/log/letsencrypt \
  certbot/dns-cloudflare:latest certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /var/lib/letsencrypt/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 30 \
  -d "*.subdomain.domain.com" -d "subdomain.domain.com" \
  --non-interactive --agree-tos --email your-email@example.com
```

6. If successful, you'll find your new certificates at `.../docker/certbot/etc_letsencrypt/live`. You'll need to download the following files:
 - Private Key - `privkey.pem`
 - Certificate - `cert.pem`

7. Import the certificate to your NAS. It'll be in Control Panel > Security > Certificate. Select Add > Add a new certificate > Import certificate > Upload the `Private Key` and `Certificate`. Don't worry about the Intermediate certificate. 

8. Your certificates page should now look something like this: 
![brave_kpcgyR2F1v](https://github.com/user-attachments/assets/d86cb779-48f8-4c85-9345-b12f1e607d88)

## Configure the certificate check script

Download the certificate check script and run it once to create a configuration file. Then, add your certificate details to the file.

1. Download the script and make it executable:

    ```bash
    sudo wget -O check_certs.sh https://raw.githubusercontent.com/telnetdoogie/synology-scripts/main/check_certs.sh && \
    sudo chmod +x check_certs.sh
    ```
2. Run the script to create the configuration file:

    ```bash
    sudo ./check_certs.sh
    ```

3. Open the configuration file in a text editor and add your certificate's Common Name (CN) and file path: 

    ```json
    {
      "config": [
        {
          "cn": "*.subdomain.domain.com",
          "cert_path": "/volume1/docker/certbot/etc_letsencrypt/live/domain.com"
        }
      ]
    }
    ```

## Schedule certificate renewals

Let's Encrypt certificates expire every 90 days, but you can automate renewals with the Synology Task Scheduler. To do this, schedule two tasks: one to renew your certificates and another to install them using `check_certs.sh`. 

1. In Synology DiskStation Manager, go to **Control Panel** > **Task Scheduler**.

1. Using **Create** > **Scheduled Task** > **User Defined Script**, add two repeating tasks which run as `root` one hour apart.

    For each task, in the **Task Settings** tab under **Run Command**, enter the following.

    a. For the script to **renew certificates** in the Certbot folder:

    ```bash
    /bin/bash
    sudo docker run -v /volume1/docker/certbot/etc_letsencrypt:/etc/letsencrypt \
        -v /volume1/docker/certbot/lib_letsencrypt:/var/lib/letsencrypt \
        -v /volume1/docker/certbot/logs:/var/log/letsencrypt \
        --rm \
        --cap-drop=all \
        certbot/dns-cloudflare:latest \
        renew
    ```
    b. For the script to **install certificates** on your NAS:

    ```bash
    /bin/bash
    cd /path/to/script # Change into the script directory 
    bash /path/to/script/check_certs.sh --update
    ```

## Adding Cloudflare CNAME wildcard
Now that we've got a wildcard SSL certificate for our subdomain, we'll need to add a couple of CNAME records which will point our domain towards our Tailnet name. 
1. The first record we'll add will have the following attributes:
    - Type: `CNAME`
    - Name: `*.subdomain` (a subdomain of your choosing)
    - Target: `ds920.tailb12345.ts.net` (where `tailb12345` is your tailnet name and `ds920` is the name of your NAS)
    - Proxy status: `DNS Only`
    - TTL: `Auto`
2. The second is optional, but I like to have this as a kind of "home page" where I can link all of my services in one place.
    - Type: `CNAME`
    - Name: `subdomain`
    - Target: `ds920.tailb12345.ts.net`
    - Proxy status: `DNS Only`
    - TTL: `Auto`

And that's Cloudflare setup complete! Your DNS records page should look something like this:
![brave_KEza5JvOuB](https://github.com/user-attachments/assets/210d9017-1b49-4a80-a3d4-19fc3138ecee)

## Synology NAS reverse proxy
Since we've only set up a wildcard CNAME record, we still need to tell the NAS where to route a request. One of the amazing showstopping spectacular features of Synology DSM is a built-in reverse proxy manager that's ready to go. We'll be using this to point various sub-subdomains of our choosing to the correct service. In this example, we'll add a reverse proxy to connect to DSM.
1. Open Control Panel > Login Portal > Advanced > Reverse Proxy
2. Hit Create
3. Under General:
    - **Source**:
    - Name: `dsm`
    - Protocol: `HTTPS`
    - Hostname: `dsm.subdomain.domain.com`
    - Port: `443`
    - HSTS: Unchecked
    - **Destination**:
    - Protocol: `HTTPS` (because we'll be connecting to the secure port 5001)
    - Hostname: `localhost`
    - Port: `5001`
4. Hit Save
5. Add as many services as you desire. **Important: you should only use HTTPS in the destination if you are positive that it's a secure port. If you aren't sure, use HTTP. In most cases, you will be using HTTP.**
Once configured, your reverse proxy list should look something like this:
![brave_I4qRUgBoUc](https://github.com/user-attachments/assets/0d3ae71a-1df8-4d0f-ad46-33e4ac79ca91)
6. Now, open back up your certificate settings and set your reverse proxies to use your wildcard SSL certificate:
![brave_YIf2Arwvmm](https://github.com/user-attachments/assets/ba08973f-6602-427d-89e4-10a86192024e)

## Testing
If we've done everything correctly, we should be able to get to each of our services securely (without having to worry about ports!) Let's give it a go with Portainer:
<img width="1470" alt="Screenshot 2024-12-27 at 11 18 19" src="https://github.com/user-attachments/assets/081ff9cc-f430-4d5c-a6a7-008e349668a8" />
Success! 

> Disclaimer: Some of this guide was copied verbatim from https://github.com/btbristow/tutorials/blob/main/certbot-with-cloudflare.md
