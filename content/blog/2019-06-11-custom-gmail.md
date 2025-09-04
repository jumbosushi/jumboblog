+++
title = "Build a custom email with Mailgun & Namecheap on Gmail"
date = "2019-06-11"
slug = "build-custom-email-with-mailgun-and-namecheap"
+++

![title logo](images/2019-06-11-custom-email/title_logo.png)

I've used Zoho Mail for quite some time but was never happy with their UI. With some googling, I found that [Mailgun](https://www.mailgun.com) makes it pretty easy for me to set up a custom email that I can manage on Gmail with my own domain from [Namecheap](https://www.namecheap.com) for free!

In this tutorial, we'll go over how I created my own personal email (`atsushi@yatsushi.com`) using the Namecheap domain I purchased (`yatsushi.com`). We will create a new receiving and sending route specifically for our new email, which will then be forwarded to an existing Gmail address. We'll also add a new account on Gmail where we can send or reply to emails straight from Gmail!

## Registering a domain on Mailgun

First, go to the [Mailgun](https://www.mailgun.com) website and create an account. Mailgun has a free plan for **up to 10,000 emails per month** as of June 2019, and if you ask me, that's more than enough for personal use.

After you've logged in, go to **Sending > Domains** from the sidebar. Click on "Add New Domain" from that page (aka [here](https://app.mailgun.com/app/sending/domains/new)).

Here's what it might look like:

![Add domain](images/2019-06-11-custom-email/add_domain.png)

Although they recommend using a dedicated subdomain for email, using the main domain as `yatsushi.com` worked for me.

When you add a new domain, the page will jump to the DNS settings page. Leave this page open in one tab, and open a new tab with Namecheap to do the next step simultaneously.

## Adding DNS records

Jump to your domain's DNS settings page on Namecheap. Before we put in any new DNS records, let's check what Mailgun says to do.

![Mailgun setting](images/2019-06-11-custom-email/mailgun_setting.png)

(Grayed out my own DNS record values)

**DO NOT copy Mailgun's hostname field into Namecheap's "Host" value!** (I waited 48 hours until I realized I did something wrong :/). It turns out Namecheap expects its own domain to be written as an **@** sign. This means where Mailgun expects the `mailo._domainkey.example.com` field, remove `example.com`, so on Namecheap you would write `mailo_domainkey` only. After you fill it out this way, the form should look something like this:

![Namecheap setup](images/2019-06-11-custom-email/namecheap_setup_1.png)

Do the same operation for MX Records and CNAME! (The MX record's host should be **@**, and the CNAME record's host field would be written as `email` instead of what Mailgun says to write, which is `email.example.com`)

Now that DNS records are added, let's refresh the **Domain Settings** page a couple of times. Note that Namecheap claims DNS record updates sometimes take a couple of hours to 48 hours (for me, it took only a couple of minutes).

If everything is working on the Namecheap side at this point, you should see a green checkmark along with the different requirements.

![Complete setup page](images/2019-06-11-custom-email/complete_setup_page.png)

## Setting up a receiving route

We're ready to set up a receiving route on Mailgun. Go to the **Receiving** settings page from the sidebar, and click on **Create Route** (aka [here](https://app.mailgun.com/app/receiving/routes/new)). This page is where we make our new email forward to an existing Gmail address.

I decided to use the "Match Recipient" option, but "Catch All" should work as well. For the recipient, we'll put our new email (`atsushi@yatsushi.com` for me). For the **Forward** section, we'll put our existing Gmail address. Here's what it might look like when this part of the page is done:

![Receiving route setup](images/2019-06-11-custom-email/receiving.png)

Set the priority to a low number (I use 1), and we're good to go! Click "Create Route", and we'll move on to the sending part.

## Setting up a sending route

Go to **Sending > Domain Settings** from the side menu. Find the **SMTP Credentials** tab header. Once you're on this page, click on "New SMTP User". This is where we get to define our new custom email! Very exciting. Go ahead and create new credentials.

![SMTP credentials](images/2019-06-11-custom-email/receive_route.png)

It will give you a password, and **be sure to store it somewhere safe since they won't show it to you again!** We'll be using the other details included on this page to set up a new account on Gmail.

## Adding a new account on Gmail

Go to your Gmail inbox, and go to the **Settings** page from the gear icon. Click on the **Accounts and Import** tab, and go to the **Add another email address** link from the *Send mail as* subsection.

![Add email](images/2019-06-11-custom-email/add_email.png)

In the new popup, put your new address into the *email* field, then click next. On the next page, we'll put what the SMTP Settings page on Mailgun suggested. Use **smtp.mailgun.org**, **your new email**, and the **previously saved password** in each field. Here's what it may look like:

![SMTP settings](images/2019-06-11-custom-email/smtp.png)

When you click **Add Account**, Gmail will send a confirmation email to the new email **which should be forwarded to your own Gmail address**! Complete the confirmation steps, and Gmail is set up to send new emails from your custom domain email!

![New email compose](images/2019-06-11-custom-email/new_email.png)

And that's it! Your email is ready to be used.

Feel free to email me at `atsushi@yatsushi.com` with your brand new email to let me know it works, or if you have any feedback on this post!
