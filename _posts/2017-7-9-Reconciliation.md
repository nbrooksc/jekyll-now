---
layout: post
title: Reconciling Inventory on 3dCart and Amazon MWS
---

I recently had the opportunity to build an inventory reconciliation app. Here's the basic idea- an e-commerce entrepreneur has a presence on 3dCart and Amazon. Until recently, he was tracking his inventories manually. Once a week, he would go through each website, writing down the inventory for each one of his dozens of products, most of which have different sizes. If there was an inconsistency, he'd need to figure out which number accurately reflected his stock. This was a time consuming and error prone process. 

The stakes are high here. You don't want to sell a product that you don't have, and you also don't want to miss an opportunity to sell a product that you do have. 

I've never looked at the APIs for Amazon or 3dCart, so this was a learning experience for me. I found the documentation to be okay. It was definitely good to get started, but some of the functionality was less than clear to me. As always, I recommend you start by looking over the documentation. 

[3dcart API](https://apirest.3dcart.com)

[MWS API](https://developer.amazonservices.com)

Note that the Amazon API is referred to as MWS, for marketplace web service. This is to differentiate it from the ten thousand other things that Amazon does.

I also didn't find much information online, which is why I'm writing this post. Hope it helps people. 

One of the main differences here is that when you're using the 3dCart API, you can query it live. This is a positive in that it's a more straightforward approach, and there are fewer steps. The downside is that if the end goal is to create a live website that your client can access at any time, this querying process will cause the page to load slowly. For me it's probably around a half minute, your mileage may vary.

Fortunately (for me at least!) for this project, the end goal was a daily email report, so the slow load time isn't relevant. 

Quick side note on that the PHP mail function

```php
mail($to, $subject, $message, $headers);
```

wasn't working for me for some reason. Working on an AWS server, I found it eaiser to use Amazon's SES (simple email service.) 

```php
$headers = array (
  'From' => SENDER,
  'To' => RECIPIENT,
  'Subject' => SUBJECT,  
  'Content-type' => "text/html;charset=UTF-8",
  'MIME-Version' => "1.0");

$smtpParams = array (
  'host' => HOST,
  'port' => PORT,
  'auth' => true,
  'username' => USERNAME,
  'password' => PASSWORD
);

 // Create an SMTP client.
$mail = Mail::factory('smtp', $smtpParams);

// Send the email.
$result = $mail->send(RECIPIENT, $headers, BODY);

if (PEAR::isError($result)) {
  echo("Email not sent. " .$result->getMessage() ."\n");
} else {
  echo("Email sent!"."\n");
}
```

The straightforward 3dCart system contrasts with the MWS system, which requires three separate steps to access the database. Fortunately, all of these steps follow the same basic syntax. You'll need an array to contain the parameters. 

```php
$param = array();
```
You can then populate the array with the relevant information based on the action that you're performing

One - Request a Report

```php
$param['Action'] = 'RequestReport';
$param['ReportType'] = '_GET_MERCHANT_LISTINGS_DATA_BACK_COMPAT_';
```
 
Two - Request a list of Reports-This list will (after a few seconds) include the report that you just requested. The list will include the Report's ID

```php
$param['Action'] = 'GetReportList';
$param['ReportTypeList.Type.1'] = '_GET_MERCHANT_LISTINGS_DATA_BACK_COMPAT_';
$param['ReportTypeList.Type.2'] = '_GET_AFN_INVENTORY_DATA_';
$param['ReportTypeList.Type.3'] = '_GET_FBA_MYI_UNSUPRRESSED_INVENTORY_DATA_';
```


Three - Use the Report ID that you received in step 2 to Access the Report

```php
$param['Action'] = 'GetReport';
$param['ReportId'] = $reportId;
```
<br />

Note that you can use the Amazon system to schedule when reports will be generated. 

```php
$param['Action'] = 'ManageReportSchedule';
$param['ReportType'] = '_GET_FBA_MYI_UNSUPRRESSED_INVENTORY_DATA_';
$param['Schedule'] = '_12_HOURS_';
```

For my particular situation, it's enough to create a new report every twelve hours. Then, every time an email needs to be sent out (or the website is accessed), a script (list.php) is run that completes step 2, and parses out the most recent relevant report ID. 

To parse out the the relevant report ID, you should first divide the response from the server into different an array of the reports 

```php
$lines = explode("\n", $response);
```

Then define Constants that contain the keywords that identify the reports that you are looking for

```php
define("MMARKER", "MERCHANT");
define("AMARKER", "AFN");
define("UMARKER", "MYI");
```

So many caps, I'm not yelling at you! 

You can then use PHP's strpos to see if a line contains the relevant marker. If it does, you can extract the ID.

```php
if (strpos($lines[0], MMARKER) !== false) {
	$mReport = $lines[2];
	$mDate = $lines[4];
} // else ... repeat as necessary 
```

Note that I'm also grabbing the date of the report, so the user will know when the inventories were last updated.

This then allows another script (getAmazon.php) to access the report and create an Array of items with their relevant inventories. Again, first create an array of the different lines from the response. Also create an array for storing the products.

```php
$splitContents = explode("\n", $response);
$amazonArray = array();
```

Loop through the array. The information is tab delimited, so you can use explode to get an array of the different columns

```php
foreach ($splitContents as $line) {
	$bits = explode("\t", $line);
```

The $amazonArray is set up where the key is the SKU and the value is the quantity. The SKU is the zero item of the array, where the quantity is in slot five. I used trim for the quantity to remove any whitespace. 

```php
$steve = $bits[0];
$martin = trim($bits[5]);
$amazonArray[$steve] = $martin;

// close the foreach loop
}
```

One thing that made this project tricky for me was matching the serial numbers. 3dCart has a parent SKU system-Let's say you have a shirt, and its part # is ABC-1852-RA. This 'parent SKU' can then be broken down into part numbers by inserting the size of the different shirts, leaving you with part numbers that look like ABC-1852-SM-RA or ABC-1852-XL-RA. 

Seems logical, right? The problem for me was that it wasn't consistent. One product might insert a hyphen before the size, while another may not. The size would usually be the penultimate piece of information in the string, except that sometimes it wasn't. I ended up creating a gauntlet of functions to push the part # number through to match it to its parent SKU.

For example- 

```php
function digitStrip($partNumber) {
	// Array of characters. By removing these characters, 
	// we may be able to get the part# to equal the SKU
	
	$bad = array("l", "L", "m", "s", "S", "M", "X", "x");
	$newName = str_replace($bad, "", $partNumber);
	
	return $newName;
}
```

You can see the rest in the functions.php script in the repository 

Matching the Part numbers with the Parent SKUs was necessary to create a logical and user friendly interface. 

I wasn't able to get the relevant information from the standard 3dCart Rest API. This was disappointing to me. I could get the Product Name and the Quantity, but it was divided by Parent SKU. I don't know who this would be useful to-It doesn't help to know that you have 8 Black shirts in stock if you don't know how many are medium and how many are large.

To solve this issue I turned to the advanced, or SOAP API. I wasn't able to find much information on this online, it looks to me that you can still use it but they don't really advertise it. It kind of felt like ordering off the secret menu at a fast food place.

```php
// Create array for relevant information and
// Connect through soap

$params = array(
	'storeUrl' => 'your.secure-url.here',
	'userKey' => 'your unique user key here'
);

$db = new soapclient('http://api.3dcart.com/cart_advanced.asmx?WSDL', array('trace', => 1, 'soap_version' => SOAP_1_1));
```

Quick note - Don't take it for granted that your PHP Installation has soap installed. If it doesn't, you won't be able to call the soapclient() function. You package manager (yum, Entropy, etc) should be able to resolve any issue. 

You need to be careful here because the SOAP API gives you direct access to your client's database. This is very powerful. At the same time, using this API will be very straightforward provided that you're familiar with SQL syntax. 

```php
// Retrieve the product list
$result = $db->runQuery($params + array(
  'sqlStatement' => "SELECT * FROM options_Advanced"
));
```

Information on the Soap API  was difficult for me to find, so let me share it here. 

[3dCart tables](https://drive.google.com/file/d/0B4LWoAow1QGLX3BuWUphSkNpTWc/view)

[3dCart Table Structure](https://drive.google.com/file/d/0B4LWoAow1QGLWmhtSjRVNjIwS00/view)

For me, the options_Advanced table ended up being the most useful. Using this table I was able to get every part number and its corresponding inventory. 

Once you've run the query, you'll have an XML result. I have more experience dealing with JSON so this took a minute for me to grok. 

Note that the array follows the same structure (SKU for Key, quantity for Value) as the Amazon Array

```php
  $sxe = new SimpleXMLElement($result->runQueryResult->any);

  // array for storing the result
  // key = part# value = quantity 
  $cartArray = array();

  // Cycle through result, grab relevant metrics
  // Throw them into array
  if (!empty($sxe->runQueryRecord)) {
    foreach ($sxe->runQueryRecord as $record) {
      $part = (string) $record->AO_Sufix;
      $quantity = (string) $record->AO_Stock;
      $part = trim($part);
      $cartArray[$part] = $quantity;
    }
  }
```


I still needed to query the REST API to get the product name. Without the product name the email report would just be a a list of serial numbers-not very useful! 

Amazon was generally more straightforward to me. The biggest thing here is to think about the report that you actually need. Be aware that Amazon divides inventory into a handful of categories- MFN fulfillable, AFN Warehouse, AFN Fulfillable, etc. If you have an Amazon store, this may be intuitive to you, but it took me a minute to grok it. If you're not careful, what could end up happening is that you could miss out on items without knowing it. Your client may have most of her inventory in the fulfilled by Amazon network, but then a couple items could still end up hanging out in the fulfilled by Merchant column. My two recommendations here-

1. Talk to your client about how he or she uses Amazon and where the inventory is. 
2. Use the _GET_FBA_MYI_UNSUPPRESSED_INVENTORY_DATA_ report. This single report contained all of the inventory from the various categories. 

So now each site has an array with the key being the part number and the value being the inventory for that site.

We can now combine these arrays into an object. item.php defines this class- there are properties for the part number, the cart inventory total, the FBA inventory total, and the merchant fulfilled inventory. Note that there is also a function to return a boolean based on whether or not the inventories match up. 

```php
<?php # item.php
      # create class for storing 
      # products
  
  class item {
    var $partNumber;
    var $fInventory;
    var $cInventory;
    var $mInventory;
    
    function __construct($pNumber, $fIn, $cIn, $mIn) {
      $this->partNumber = $pNumber;
      $this->fInventory = $fIn;
      $this->cInventory = $cIn;
      $this->mInventory = $mIn;
    }
  function isEven() {
      $amazonInventory = 0;
      if (is_numeric($this->fInventory)) {
        $amazonInventory += $this->fInventory;
      }

      if (is_numeric($this->mInventory)) {
        $amazonInventory += $this->mInventory;
      }

      // Adding step to remove strange -1 inventory
      // from 3dCart

      $cartIn = ($this->cInventory > 0) ? $this->cInventory : 0;


      if ($amazonInventory == $cartIn) {
        return true;
      }

      return false;
    }
  }
?>
```

Using the $nameArray from the 3dCart REST API, we can then create an array of these items where the key is the product name. 

```php
 // Array to contain all of the products
  // key will be the name, value will be
  // an object that contains part # and stock info 
  $itemsArray = array();

  // initialize each element of the array
  forEach ($nameArray as $one=> $two) {
    $itemsArray[$two] = array();
  }
```

I created one site, reconcile3.php, that prints the entire array in HTML tables. This was only used for me for debugging. 

```php
// display array using HTML tables

foreach($itemsArray as $one => $two) {

    // check to see if there are any
    // actual products associated with the name    

    if (!empty($two)) {
      echo "<br><br>";
      echo "<table>";
      echo "<caption><h2>$one</h2></caption>";
      echo "<tr>";
      echo "<th>Part #</th>";
      echo "<th>3dCart Inventory</th>";
      echo "<th>FBA Inventory</th>";
      echo "<th>Amazon Merchant Inventory</th>";
      echo "</tr>";

      foreach($two as $three => $four) {
        $rDub = $four->get_part();
        $fIn = $four->get_f();
        $cIn = $four->get_c();
        $mIn = $four->get_m();

        echo "<tr>";
        echo "<td style='text-align:left;'>$rDub</td>";
        echo "<td>$cIn</td>";
        echo "<td>$fIn</td>";
        echo "<td>$mIn</td>";
        echo "</tr>";
      }
      echo "</table>";
    }
  }
```

The important site is uneven.php, which displays only the products with uneven inventories. 