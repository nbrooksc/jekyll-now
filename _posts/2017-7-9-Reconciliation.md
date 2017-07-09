---
layout: post
title: Reconciling Inventory on 3dCart and Amazon MWS---

adventerra blog post

I recently had the opportunity to build an inventory reconciliation app. Here's the basic idea- an e-commerce entrepreneur has a presence on 3dCart and Amazon. Until recently, he was tracking his inventories manually. Once a week, he would go through each website, writing down the inventory for each one of his dozens of products, most of which have different sizes. If there was an inconsistency, he'd need to figure out which number accurately reflected his stock. This was a time consuming and error prone process. 

The stakes are high here. You don't want to sell a product that you don't have, and you also don't want to miss an opportunity to sell a product that you do have. 

I've never looked at the APIs for Amazon or 3dCart, so this was a learning experience for me. I found the documentation to be okay. It was definitely good to get started, but some of the functionality was less than clear to me. As always, I recommend you start by looking over the documentation. 

3dcart api - https://apirest.3dcart.com

https://developer.amazonservices.com

Note that the Amazon API is referred to as MWS, for marketplace web service. This is to differentiate it from the ten thousand other things that Amazon does.

I also didn't find much information online, which is why I'm writing this post. Hope it helps people. 

One of the main differences here is that when you're using the 3dCart API, you can query it live. This is a positive in that it's a more straightforward approach, and there are fewer steps. The downside is that if the end goal is to create a live website that your client can access at any time, this querying process will cause the page to load slowly. For me it's probably around a half minute, your mileage may vary.

Fortunately (for me at least!) for this project, the end goal was a daily email report, so the slow load time isn't relevant. 

This contrasts with the MWS system, which requires three separate steps to access the database. What you need to do is 

1. Request a Report 
2. Request a list of Reports-This list will (after a few seconds) include the report that you just requested. The list will include the Report's ID
3. Use the Report ID that you received in step 2 to Access the Report

Working on a Linux system, my approach was to make a cron for Step 1. For my particular situation, it's enough to create a new report every twelve hours. Then, every time an email needs to be sent out (or the website is accessed), a script (list.php) is run that completes step 2, and parses out the most recent relevant report ID. This then allows another script (getAmazon.php) to access the report and create an Array of items with their relevant inventories. 

One thing that made this project tricky for me was matching the serial numbers. 3dCart has a parent SKU system-Let's say you have a shirt, and its part # is ABC-1852-RA. This 'parent SKU' can then be broken down into part numbers by inserting the size of the different shirts, leaving you with part numbers that look like ABC-1852-SM-RA or ABC-1852-XL-RA. 

Seems logical, right? The problem for me was that it wasn't consistent. One product might insert a hyphen before the size, while another may not. The size would usually be the penultimate piece of information in the string, except that sometimes it wasn't. I ended up creating a gauntlet of functions to push the part # number through to match it to its parent SKU.

Matching the Part numbers with the Parent SKUs was necessary to create a logical and user friendly interface. 

I wasn't able to get the relevant information from the standard 3dCart Rest API. This was disappointing to me. I could get the Product Name and the Quantity, but it was divided by Parent SKU. I don't know who this would be useful to-It doesn't help to know that you have 8 Black shirts in stock if you don't know how many are medium and how many are large.

To solve this issue I turned to the advanced, or SOAP API. I wasn't able to find much information on this online, it looks to me that you can still use it but they don't really advertise it. It kind of felt like ordering off the secret menu at a fast food place.

You need to be careful here because the SOAP API gives you direct access to your client's database. This is very powerful. 

This was difficult for me to find, so let me share it here. 

3dCart tables - https://drive.google.com/file/d/0B4LWoAow1QGLX3BuWUphSkNpTWc/view

3dCart Table Structure - https://drive.google.com/file/d/0B4LWoAow1QGLWmhtSjRVNjIwS00/view

For me, the options_Advanced table ended up being the most useful. Using this table I was able to get every part number and its corresponding inventory. 

I still needed to query the REST API to get the product name. Without the product name the email report would just be a a list of serial numbers-not very useful! 

Amazon was generally more straightforward to me. The biggest thing here is to think about the report that you actually need. Be aware that Amazon divides inventory into a handful of categories- MFN fulfillable, AFN Warehouse, AFN Fulfillable, etc. If you have an Amazon store, this may be intuitive to you, but it took me a minute to grok it. If you're not careful, what could end up happening is that you could miss out on items without knowing it. Your client may have most of her inventory in the fulfilled by Amazon network, but then a couple items could still end up hanging out in the fulfilled by Merchant column. My two recommendations here-

1. Talk to your client about how he or she uses Amazon and where the inventory is. 
2. Use the _GET_FBA_MYI_UNSUPPRESSED_INVENTORY_DATA_ report. This single report contained all of the inventory from the various categories. 

Also note that the Amazon Reports are tab delimited. For PHP, the easiest way for me to parse this was to explode by \n to get the different entries, then to explode the entries by \t to get the columns. 

So now each site has an array with the key being the part number and the value being the inventory for that site.

We can now combine these arrays into an object. item.php defines this class- there are properties for the part number, the cart inventory total, the FBA inventory total, and the merchant fulfilled inventory. Note that there is also a function to return a boolean based on whether or not the inventories match up. 

Using the $nameArray from the 3dCart REST API, we can then create an array of these items where the key is the product name. 

I created one site, reconcile3.php, that prints the entire array in HTML tables. This was only used for me for debugging. 

The important site is uneven.php, which displays only the products with uneven inventories. 