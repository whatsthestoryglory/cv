---
title: "Scraping product prices from the internet"
date: 2021-08-29T11:29:25-05:00
description: Building a web scraper for popular e-commerce sites
hero: /images/posts/writing-posts/scraping.svg
menu:
  sidebar:
    name: E-Store Scraping
    identifier: ctscraper
    weight: 100
---

All summer long I have wanted to purchase a new grill. I did a ton of research, looking for the perfect combination of size, features, and longevity, and what I quickly found was that my last qualifier - longevity - was one of the most expensive qualities to buy for. Quality grills are _expensive_ at current regular prices, perhaps I needed to wait for a sale. I just needed to find one.

Many people are familiar with sites like [camelcamelcamel](https://www.camelcamelcamel.com) that provide historic prices for Amazon products, but what if I don't want to buy from Amazon? There are surprisingly few options for monitoring prices from other retailers, especially those that serve the Canadian market. What I could find had limited functionality and wouldn't do what I needed. I decided to create something for myself.

## Scraping a single product page

Building a web scraper is a common task with numerous how-to documents and videos around the internet supporting a variety of languages. I thought this meant my project would be relatively easy to build and get running, if only for one or two retailers. Unfortunately, modern web technologies stepped in and proved to me that things were more complex than I had anticipated.

I started [here](https://blog.rstudio.com/2014/11/24/rvest-easy-web-scraping-with-r/), "rvest: Easy Web Scraping with R" the article is called. How could it not be easy? First, I pulled a product page using the `rvest` package.

```
library(rvest)
product_page <- read_html("https://www.canadiantire.ca/en/pdp/shark-rocket-ultra-light-corded-stick-vacuum-0436795p.html")
```

Unfortunately for me, this didn't work. The session hung and no HTML was read. What happened?

### Diagnosing the Problem

With any Programming problem (really, most problems), I like to start by going step by step through the process that encountered the problem. After a little bit of research, I found that `rvest` uses `httr`, and that `httr` uses `libcurl` to perform requests. I figured I'd go straight to the source and try using `libcurl` directly from the command line.

```
curl https://www.canadiantire.ca/en/pdp/shark-rocket-ultra-light-corded-stick-vacuum-0436795p.html
```

I got a response this time...sort of.

```
<HTML><HEAD>
<TITLE>Access Denied</TITLE>
</HEAD><BODY>
<H1>Access Denied</H1>

You don't have permission to access "http&#58;&#47;&#47;www&#46;canadiantire&#46;ca&#47;en&#47;pdp&#47;shark&#45;rocket&#45;ultra&#45;light&#45;corded&#45;stick&#45;vacuum&#45;0436795p&#46;html" on this server.<P>
Reference&#32;&#35;18&#46;95973017&#46;1631193900&#46;91d69
</BODY>
</HTML>
```

Now that's interesting. Why do I get 'Access Denied' from `curl` but I can still view the site in my browser? Something else is going on.

### User Agents

What is a user agent? When you visit a website, the user agent is a string that your browser sends to the website to tell it what kind of software you're running. This allows the website to offer different results based on your equipment and software, helping to eliminate compatibility issues. When I use `rvest` or `curl` this program name is reflected in the user agent sent to the website, and the website sees this and returns an "Access Denied" error back to me. I guess they only want browsers they're familiar with visiting the store.

Conveniently, curl offers a way to manually define what user agent is sent. I then tried setting the user agent to mimic that of a Windows 10 PC accessing the site through Firefox.

```
curl https://www.canadiantire.ca/en/pdp/shark-rocket-ultra-light-corded-stick-vacuum-0436795p.html -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:91.0) Gecko/20100101 Firefox/91.0"
```

Success! I now saw the page returned by `curl`. Now we just need to apply the same process back to my `R` code. Unfortunately, the `rvest` package doesn't have a function to set the user agent, but `httr` does.

```
product_response <- GET("https://www.canadiantire.ca/en/pdp/shark-rocket-ultra-light-corded-stick-vacuum-0436795p.html", user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:91.0) Gecko/20100101 Firefox/91.0")
product_page <- read_html(product_response)
```

Perfect, now it worked, I have an html page loaded in to the variable `product_page`.

### Selecting a field

Within the `rvest` package is the `html_nodes` function which allows you to search through the HTML and look for specific fields from their CSS.

To find what field I would need to search for, I poked through the site using the [SelectorGadget](https://selectorgadget.com/) plugin for Chrome. I found the class `.price__reg-value_multisku` contained the price, so I continued to follow the article and modified their code to pull the price from the html.

```
product_page %>%
  html_nodes(".price__reg-value_multisku") %>%
  html_text()

#> character(0)
```

What? I was sure I did everything right this time, what happened??

I have to admit, this one took me a while to figure out. I needed to save the html file and sift through it manually before it made sense. There was no `.price__reg-value_multisku` in the HTML I saved! I loaded the page again into my browser and opened the source code to compare, and it was there, but it wasn't in the code I downloaded with R. What was going on?

## Rendering the Page

Many, if not most, of today's websites have some form of dynamic content on them. This could be showing you the store closest to you, updating the page to have a darker background when viewed in the evening, or even changing the price depending on your location. What I found when looking through the html file I saved was that in this case, all of the product information was loaded dynamically. The HTML my browser was sent would include only the barebones of the page, and then there was some `JS` scripting that would go on to request the specifics of the product separately and then render them. All of this happens so quickly in the browser that you may not even notice, but unfortunately my method of getting the page did notice.

### The magic of PhantomJS

[PhantomJS](https://phantomjs.org/) is a "Scriptable Headless Browser." Basically what that means is that it does the things your regular web browser would do, but without needing a screen to display it on. You can then write scripts telling it to do things with these results, such as capturing a screenshot of the fully rendered page, or saving the final HTML once all javascript has run.

I found [this blog](https://velaco.github.io/how-to-scrape-data-from-javascript-websites-with-R/) that gave me an example using PhantomJS to render the page and store the result as a .html file. I then ran their code, modified to use my own link, and saved the page to `ctire.html`.

```
system("phantomjs.exe ctscrape.js")
product_page <- read_html("ctire.html")
product_page %>%
  html_nodes(".price__reg-value_multisku") %>%
  html_text()

#> character(0)
```

Even after all this, I still have no price! What went wrong now?

### Making PhantomJS do my bidding

The tricky thing with PhantomJS is timing. It's not just about _what_ you want it to do, but _when_. It's not always easy to know when a web page has "finished loading," so determining when to save the resulting html file can be a challenge.

After some searching, I came across a solution on [Github](https://gist.github.com/th3d0g/11234133). By implementing this `waitFor` function, I could tell PhantomJS to wait for the price information to be rendered before saving the file. Below is the line that covers the element to wait for;

```
return $("#signin-dropdown").is(":visible");
```

In this example, you can see that we want the CSS element with the ID `signin-dropdown` to have the property of `visible`. For my purpose, I changed this to look for the class ID `.pdp-buy-box__primary-section` and re-ran the test.

```
system("phantomjs.exe ctscrape.js")
product_page <- read_html("ctire.html")
product_page %>%
  html_nodes(".price__reg-value_multisku") %>%
  html_text()

#> "$149.99"
```

Success! I can now scrape the price given a Canadian Tire product URL. In my next post, I'll talk about saving this result using [MongoDB](https://www.mongodb.com) and the power you can weild from there. Feel free to follow along on [my github](https://github.com/whatsthestoryglory/canada-price-tracker)
