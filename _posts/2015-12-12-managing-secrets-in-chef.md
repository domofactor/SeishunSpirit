---
published: false
layout: post
title: Managing Secrets in Chef
tags: 
  - chef
  - secrets
  - encrypted databags
  - databags
  - s3
  - citadel
categories: 
  - Cars
excerpt: "My thoughts on Encrypted Databags, Chef Vault and how to deal with secrets in Chef"
comments: true
share: true
---

When I first started cooking with Chef, I was curious about built-in solutions for dealing with secret stuff like passwords and keys. Turns out, there are two options, Encrypted Databags and Chef Vault. 

After poking around, I was basically told by my DevOps Teammate:
![](http://i.imgur.com/MJAkqK8.jpg)

Chef Vault is built on Encrypted Databags, the same also holds true there.

Lets ask some questions to get some answers on what's going on here.

##What are databags?
Databags are variables that get stored on the Chef Server as JSON data.

##What are common methods of encryption?
###Symmetric Encryption
Uses the same key to both Encrypt and Decrypt your secret
![](http://media.packetlife.net/media/blog/attachments/512/symmetric_encryption.png)

###Asymmetric Encryption
Uses 2 seperate keys. The public key is used to encrypt the secret and a private key to decrypt the secret.
![](http://media.packetlife.net/media/blog/attachments/511/asymmetric_encryption.png)

##What are encrypted databags?
Databags that are encrypted using Asymmetric encryption so your secrets can be stored in version control(Github, Bitbucket, ect) safely.

##How do encypted databags Work?
Now that we have a basic understanding of things, lets walkthrough how encrypted databags work.

In a typical setup, you have a Chef Server and a Chef Client.
"picture of chef server and client"

A public key is used to encrypt the databag so it can be stored in version control, and a separate private key is used by the Chef Client to decrypt the databag.

##What is bad with encrypted databags?
This seems like a pretty simple plan, but now you just turned your encryption issue into a key management one. How will we manage N number of keys for all of our databags and clients? What is the workflow like is one of our keys gets comprimised and we need to rotate when across N number of clients? 

##What is Chef Vault?
They guys over at Opscode(Chef) attempt to solve the key management problem of regular encrypted databags by using Chef Vault.

##How does Chef Vault work?
When a chef client first registers with the chef server, it creates authentication using a private/public keypair. After registration, the public key is copied over to the Chef Server for subsequent uses when running Chef Client.

Chef Vault encrypts the data it using the client's public key. Then, each client decrypts by using its own private key.

##What is bad with Chef Vault?
This overall seems like a gread solution, since it takes some pieces that Chef already has been using and helps solve the key management problem. However, in my opinion there are 2 main issue with Chef Vault.

###1. Scaling
If I have 2 clients, then encrypt my databag using their public keys. Both of my clients are happy and can decrypt with no issues.

However, if I add a 3rd client that wasnt around when I originally encryped the JSON data. That client won't be able to decrypt properly. So, now we need to re-encrypt the databag to allow the 3rd client to successfully decrypt the data.

###2. Least Privilege
If you have more than one databag, all of a sudden you have one keypair encrypting/decrypting multiple databags.

##How do we solve the problems with Chef Vault?
Until recently, in order to solve the Scaling and Least Privilege problems with Chef Vault you pretty much had to write-up your own solution. 

###1. Write your library cookbook
While I was working at [Moz](http://moz.com), we had rolled up our own solution using [Swift](https://wiki.openstack.org/wiki/Swift) and a custom library cookbook.

###2. Use Citadel or some other tool
Chef Contributer Noah Kantrowitz has a great [post](https://coderanger.net/chef-secrets/) over on his blog about this. He covers some currently available options and how he solved this problem by writing his own library cookbook called [Citadel](https://github.com/poise/citadel) using a combination of AWS S3 and AWS IAM Roles. At [Abeja](http://abeja.asia), we're currently using this method for dealing with secrets in Chef.

It works by using 3rd party access control(AWS IAM), and stores your data at-rest(S3) or what is similarly called Offline Encryption. You also have access to logging and version constrol with S3. So if somebody updates a secret, you have the ability to see who changed it and revert if needed.

One thing that is very important however, is to come up with a good schema when using the Citadel cookbook. 

For example, the we have the following file contains the string _secretpassword_ in S3.
`s3://mybucket/mycookbook/secret.pem`

Then, in your wrapper cookbook, include the citadel cookbook in your metadata.rb and reference it like so in a recipe:
`mycookbook::myrecipe.rb`
{% highlight ruby %}
file '/etc/secret' do
  owner 'root'
  group 'root'
  mode '600'
  content citadel["#{cookbook_name}/secret.pem"]
  action :create
end
{% endhighlight %}

