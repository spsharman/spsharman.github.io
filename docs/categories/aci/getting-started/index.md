# Getting started with ACI

**How do I get started with ACI?**

If I've heard this question once, I've heard it many (many) times from both customers and partners, and as boring as it sounds you really need to have plan before you start configuring "stuff" on your fabric.

Very broadly you need to have a plan in a few areas:

- you'll need to consider how you are going to configure the interfaces on your switches, and by this I don't mean whether you're opting for lldp over cdp, what I mean is that you need to understand how interface configuration actually works, and how this will work at scale in your environment

- you'll need to consider how you are going to configure your tenants, and therefore how you're going to connect (via L2/L3) to the workloads that you're hosting on the fabric

- you'll need to consider how you are actually going to do the configuration - are you going to use the GUI, the CLI, or the API? There is no right or wrong option, most likely you'll start off using the GUI to configure things, and then you'll gravitate towards using the API, maybe with Postman, Ansible, Terraform, or Nexus as Code

**Ok, so what will happen if you don't start out with a plan?**

Well much like anything else in life, whether you're configuring a standalone switch or router, or whether you're building a house, if you don't have some kind of a plan you'll most likely end up in a mess.

As a best practice, I always strongly suggest that you download the APIC simulator so that you can test your configuration/code in a safe place, then test you configuration/code on a test network, and then (finally) when you're 100% happy, you put your configuration/code into production.
