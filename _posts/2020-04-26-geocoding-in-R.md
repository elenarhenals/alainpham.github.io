Getting Country Codes with Geocoding
------------------------------------

When dealing with unstandardized address strings, it’s often challenging
to extract the country detail because it can present itself anywhere in
the string. In this post, I will explain how to use Google Maps
Geocoding API to get your address strings in order and to extract the
country name and code.

I am currently working on a project to help an international bank assess
the quality of customer data in wire payment messages. To give you a
better understanding of what information these messages may include, I
found this example from the SWIFT Payment Formatting Guide for Financial
Institutions.

| Tag            | Name                                                  |
|:---------------|:------------------------------------------------------|
| 20             | Transaction reference number                          |
| 23B            | Bank operation code                                   |
| 32A            | Value date / currency / interbank settled             |
| 33B            | Currency / original ordered amount                    |
| 50A, F or K    | Ordering customer (payer) or address of the remitter. |
| 52A or D       | Ordering institution (payer’s bank)                   |
| 53A, B or D    | Sender’s correspondent (bank)                         |
| 54A, B or D    | Receiver’s correspondent (bank)                       |
| 56A, C or D    | Intermediary (bank)                                   |
| 57A, B, C or D | Account with institution (beneficiary’s bank)         |
| 59 or 59A      | Beneficiary                                           |
| 70             | Remittance information                                |
| 71A            | Details of charges (OUR/SHA/BEN)                      |
| 72             | Sender to receiver information                        |
| 77B            | Regulatory reporting                                  |

For example, you can see that Tag 50 Ordering Customer, consists of a
number of data elements that are all bundled in a string of characters.
The address data is usually mapped to 3 lines, but there is no clear
structure as to wich line holds which address element and no clear
delimiters to parse the data.

There is a big issue with that because it’s tough to parse unstructured
data like this accurately, and yet it is critical for the banks’ ability
to timely detect and stop payments relating to fraud or money
laundering.

The country of the originating customer for incoming wires and the
country of the beneficiary customer for outgoing wires are critical
pieces of information in this regard. In this post, I will show you how
you can standardize unstructured address strings and extract the country
detail using geocoding with Google Maps API.

Ok. Let’s begin, shall we?

#### Step 1 - Prep your data

Review your address field(s). Make sure you have the entire address in
one string. If you need to concatenate some columns to achieve that,
such as in my case, you can use the unite() function from the tidyr
package. I would strongly recommend removing any trailing white space,
which is common in unstructured address strings. You can do it with the
base trimws() function.

For this demonstration, I put together a list of 5 addresses generated
from a Google search of the world’s best restaurants. I modified them a
bit, so they look similar to the unstructured address strings that can
come through in the wire messages.

``` r
#read you data in

addresses <- read.csv("addresses.csv")

knitr::kable(addresses)
```

| ORIGINATOR\_ADDRESS\_1                | ORIGINATOR\_ADDRESS\_2     | ORIGINATOR\_ADDRESS\_3 |
|:--------------------------------------|:---------------------------|:-----------------------|
| 18N CanalRd                           | Singapore                  | 48830                  |
| Cra. 13 \#8525 BogotáColombia         |                            |                        |
| RUSSIA 109004 G MOSKVA                | UL DOBROVOLCHESKAYA DOM 12 |                        |
| 127 Ledbury Rd, Notting H             | ill, London                | W11 2AQ                |
| Shop 4C-D Tower 1 PL/F, China HK City | 33 Canton road             | Tsim Sha Tsui          |

``` r
#concatenate the address columns with unite()
addresses <- addresses%>%
  unite("org_address", ORIGINATOR_ADDRESS_1, ORIGINATOR_ADDRESS_2, ORIGINATOR_ADDRESS_3, sep = " ")

knitr::kable(addresses)
```

| org\_address                                                       |
|:-------------------------------------------------------------------|
| 18N CanalRd Singapore 48830                                        |
| Cra. 13 \#8525 BogotáColombia                                      |
| RUSSIA 109004 G MOSKVA UL DOBROVOLCHESKAYA DOM 12                  |
| 127 Ledbury Rd, Notting H ill, London W11 2AQ                      |
| Shop 4C-D Tower 1 PL/F, China HK City 33 Canton road Tsim Sha Tsui |

#### Step 2 - Dedupe your data

Create a vector of unique addresses by using the unique() function. In
this demonstration, we only have 5 unique addresses. Still, in the real
world data such as customer information, there will always be
duplicates. Deduping the address data helps reduce the number of Google
Maps API calls.

``` r
org_address <- unique(addresses$org_address)
```

#### Step 3 - Get set up with Google Maps Geocoding API

Let’s get everything ready for our API calls. First of all, if you have
never used Google Maps API, you will have to register and obtain your
key.

1.  Go to
    <a href="https://cloud.google.com/maps-platform" class="uri">https://cloud.google.com/maps-platform</a>
2.  Click Get Started
3.  Sign in with your Google account or create one
4.  Proceed with setting up the billing. Google will give you a free
    $300 credit that should be enough for about 200,000 calls.
5.  Once signed up, go to the Maps API library and find Geocoding API
6.  Click on it and then click Enable
7.  Go to your credentials and copy your key

It’s that easy!

#### Step 4 - Calling Google Maps API

Now, we are ready for geocoding with Google Maps API!

You will need to install the ggmap package in your R Studio if you
haven’t installed it already and then register your API key. Google will
only authorize the API calls coming from registered users.

``` r
library(ggmap)

register_google(key = mykey)
```

Once the key is registered, we can start making API calls. The geocode()
function provides several outputs, as shown below. You can select
whichever is more applicable to your project.

``` r
geocode(org_address[1], output="latlon")
```

    ## # A tibble: 1 x 2
    ##     lon   lat
    ##   <dbl> <dbl>
    ## 1  104.  1.29

If you set output to “latlon,” the function will only return the
latitude and the longitude of the location.

``` r
geocode(org_address[1], output="more")
```

    ## # A tibble: 1 x 9
    ##     lon   lat type        loctype address                north south  east  west
    ##   <dbl> <dbl> <chr>       <chr>   <chr>                  <dbl> <dbl> <dbl> <dbl>
    ## 1  104.  1.29 street_add~ rooftop 18 n canal rd, singap~  1.29  1.28  104.  104.

The output option “more” returns additional information, including the
address in a standartized format which is always handy. If your goal is
to standartize addresses, you can stop here. Just put your API request
call inside a for loop to save yourself some time, as shown below.

``` r
org_address_std <- data.frame("Address_std"=NA)

for (i in 1:length(org_address)){
  address <- geocode(org_address[i], output="more", override_limit = TRUE)
  if(ncol(address)>=5){ #only proceeds if the API call was successful
  org_address_std[i,1] <- address$address
  }
}
```

Here is what we got from Google Maps.

``` r
knitr::kable(org_address_std)
```

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: left;">Address_std</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: left;">18 n canal rd, singapore 048830</td>
</tr>
<tr class="even">
<td style="text-align: left;">NA</td>
</tr>
<tr class="odd">
<td style="text-align: left;">dobrovol’cheskaya st, moskva, russia, 109004</td>
</tr>
<tr class="even">
<td style="text-align: left;">127 ledbury rd, notting hill, london w11 2aq, uk</td>
</tr>
<tr class="odd">
<td style="text-align: left;">shop 4c-d, tower 1, pl/f, china hong kong city, 33 canton road, tsim sha tsui, hong kong</td>
</tr>
</tbody>
</table>

Compare our results with the original addresses above. Looks much better
to me!

Of course, there will always be addresses that Google is not able to
identify or identifies incorrectly. It’s especially true when dealing
with international addresses, so I highly recommend reviewing your
results before proceeding to the next step. To help Google Maps do a
better job, you can clean up your address strings by removing odd
characters such as accent marks or irregular punctuation.

Let’s go back to our original addresses data set and clean up the 2nd
address from the top that wasn’t recognized by Google Maps.

``` r
addresses[2,1] <- str_remove_all(iconv(addresses[2,1], to='ASCII//TRANSLIT'), "#")

addresses[2,1]
```

    ## [1] "Cra. 13 8525 BogotaColombia  "

Now let’s see if Google Maps recognizes this address.

``` r
geocode("Cra. 13 8525 BogotaColombia", output = "more")
```

    ## Source : https://maps.googleapis.com/maps/api/geocode/json?address=Cra.+13+8525+BogotaColombia&key=xxx

    ## # A tibble: 1 x 9
    ##     lon   lat type       loctype      address            north south  east  west
    ##   <dbl> <dbl> <chr>      <chr>        <chr>              <dbl> <dbl> <dbl> <dbl>
    ## 1 -74.1  4.67 street_ad~ range_inter~ cra. 13 #85-25, b~  4.67  4.67 -74.1 -74.1

Perfect! Let’s update our address vector.

``` r
org_address <- unique(addresses$org_address)
```

Finally, the last output option we will review here is “all.” We are
going to use this option to access the country detail. Take a look at
the output below.

``` r
address_all <- geocode(org_address[1], output="all")
```

``` r
address_all
```

    ## $results
    ## $results[[1]]
    ## $results[[1]]$address_components
    ## $results[[1]]$address_components[[1]]
    ## $results[[1]]$address_components[[1]]$long_name
    ## [1] "18"
    ## 
    ## $results[[1]]$address_components[[1]]$short_name
    ## [1] "18"
    ## 
    ## $results[[1]]$address_components[[1]]$types
    ## $results[[1]]$address_components[[1]]$types[[1]]
    ## [1] "street_number"
    ## 
    ## 
    ## 
    ## $results[[1]]$address_components[[2]]
    ## $results[[1]]$address_components[[2]]$long_name
    ## [1] "North Canal Road"
    ## 
    ## $results[[1]]$address_components[[2]]$short_name
    ## [1] "N Canal Rd"
    ## 
    ## $results[[1]]$address_components[[2]]$types
    ## $results[[1]]$address_components[[2]]$types[[1]]
    ## [1] "route"
    ## 
    ## 
    ## 
    ## $results[[1]]$address_components[[3]]
    ## $results[[1]]$address_components[[3]]$long_name
    ## [1] "Singapore River"
    ## 
    ## $results[[1]]$address_components[[3]]$short_name
    ## [1] "Singapore River"
    ## 
    ## $results[[1]]$address_components[[3]]$types
    ## $results[[1]]$address_components[[3]]$types[[1]]
    ## [1] "neighborhood"
    ## 
    ## $results[[1]]$address_components[[3]]$types[[2]]
    ## [1] "political"
    ## 
    ## 
    ## 
    ## $results[[1]]$address_components[[4]]
    ## $results[[1]]$address_components[[4]]$long_name
    ## [1] "Singapore"
    ## 
    ## $results[[1]]$address_components[[4]]$short_name
    ## [1] "Singapore"
    ## 
    ## $results[[1]]$address_components[[4]]$types
    ## $results[[1]]$address_components[[4]]$types[[1]]
    ## [1] "locality"
    ## 
    ## $results[[1]]$address_components[[4]]$types[[2]]
    ## [1] "political"
    ## 
    ## 
    ## 
    ## $results[[1]]$address_components[[5]]
    ## $results[[1]]$address_components[[5]]$long_name
    ## [1] "Singapore"
    ## 
    ## $results[[1]]$address_components[[5]]$short_name
    ## [1] "SG"
    ## 
    ## $results[[1]]$address_components[[5]]$types
    ## $results[[1]]$address_components[[5]]$types[[1]]
    ## [1] "country"
    ## 
    ## $results[[1]]$address_components[[5]]$types[[2]]
    ## [1] "political"
    ## 
    ## 
    ## 
    ## $results[[1]]$address_components[[6]]
    ## $results[[1]]$address_components[[6]]$long_name
    ## [1] "048830"
    ## 
    ## $results[[1]]$address_components[[6]]$short_name
    ## [1] "048830"
    ## 
    ## $results[[1]]$address_components[[6]]$types
    ## $results[[1]]$address_components[[6]]$types[[1]]
    ## [1] "postal_code"
    ## 
    ## 
    ## 
    ## 
    ## $results[[1]]$formatted_address
    ## [1] "18 N Canal Rd, Singapore 048830"
    ## 
    ## $results[[1]]$geometry
    ## $results[[1]]$geometry$location
    ## $results[[1]]$geometry$location$lat
    ## [1] 1.286275
    ## 
    ## $results[[1]]$geometry$location$lng
    ## [1] 103.8483
    ## 
    ## 
    ## $results[[1]]$geometry$location_type
    ## [1] "ROOFTOP"
    ## 
    ## $results[[1]]$geometry$viewport
    ## $results[[1]]$geometry$viewport$northeast
    ## $results[[1]]$geometry$viewport$northeast$lat
    ## [1] 1.287624
    ## 
    ## $results[[1]]$geometry$viewport$northeast$lng
    ## [1] 103.8496
    ## 
    ## 
    ## $results[[1]]$geometry$viewport$southwest
    ## $results[[1]]$geometry$viewport$southwest$lat
    ## [1] 1.284926
    ## 
    ## $results[[1]]$geometry$viewport$southwest$lng
    ## [1] 103.8469
    ## 
    ## 
    ## 
    ## 
    ## $results[[1]]$place_id
    ## [1] "ChIJ_WZSrgsZ2jERm6wZgPqIiw0"
    ## 
    ## $results[[1]]$plus_code
    ## $results[[1]]$plus_code$compound_code
    ## [1] "7RPX+G8 Singapore"
    ## 
    ## $results[[1]]$plus_code$global_code
    ## [1] "6PH57RPX+G8"
    ## 
    ## 
    ## $results[[1]]$types
    ## $results[[1]]$types[[1]]
    ## [1] "street_address"
    ## 
    ## 
    ## 
    ## 
    ## $status
    ## [1] "OK"

You can see that this is a deeply nested list, which is going to be
challenging to work with. Luckily, there are some great packages
available in R for working with such data structures. Specifically, I am
talking about purr and tidyr that will help us extract and unnest list
elements.

#### Step 5 - Extracting the country detail

Before we can extract anything, we need to examine the structure of the
list object we are working with. As shown below, our list contains 2
main elements: results and status. The results element consists of 6
other elements with sub-elements of their own.

``` r
str(address_all)
```

    ## List of 2
    ##  $ results:List of 1
    ##   ..$ :List of 6
    ##   .. ..$ address_components:List of 6
    ##   .. .. ..$ :List of 3
    ##   .. .. .. ..$ long_name : chr "18"
    ##   .. .. .. ..$ short_name: chr "18"
    ##   .. .. .. ..$ types     :List of 1
    ##   .. .. .. .. ..$ : chr "street_number"
    ##   .. .. ..$ :List of 3
    ##   .. .. .. ..$ long_name : chr "North Canal Road"
    ##   .. .. .. ..$ short_name: chr "N Canal Rd"
    ##   .. .. .. ..$ types     :List of 1
    ##   .. .. .. .. ..$ : chr "route"
    ##   .. .. ..$ :List of 3
    ##   .. .. .. ..$ long_name : chr "Singapore River"
    ##   .. .. .. ..$ short_name: chr "Singapore River"
    ##   .. .. .. ..$ types     :List of 2
    ##   .. .. .. .. ..$ : chr "neighborhood"
    ##   .. .. .. .. ..$ : chr "political"
    ##   .. .. ..$ :List of 3
    ##   .. .. .. ..$ long_name : chr "Singapore"
    ##   .. .. .. ..$ short_name: chr "Singapore"
    ##   .. .. .. ..$ types     :List of 2
    ##   .. .. .. .. ..$ : chr "locality"
    ##   .. .. .. .. ..$ : chr "political"
    ##   .. .. ..$ :List of 3
    ##   .. .. .. ..$ long_name : chr "Singapore"
    ##   .. .. .. ..$ short_name: chr "SG"
    ##   .. .. .. ..$ types     :List of 2
    ##   .. .. .. .. ..$ : chr "country"
    ##   .. .. .. .. ..$ : chr "political"
    ##   .. .. ..$ :List of 3
    ##   .. .. .. ..$ long_name : chr "048830"
    ##   .. .. .. ..$ short_name: chr "048830"
    ##   .. .. .. ..$ types     :List of 1
    ##   .. .. .. .. ..$ : chr "postal_code"
    ##   .. ..$ formatted_address : chr "18 N Canal Rd, Singapore 048830"
    ##   .. ..$ geometry          :List of 3
    ##   .. .. ..$ location     :List of 2
    ##   .. .. .. ..$ lat: num 1.29
    ##   .. .. .. ..$ lng: num 104
    ##   .. .. ..$ location_type: chr "ROOFTOP"
    ##   .. .. ..$ viewport     :List of 2
    ##   .. .. .. ..$ northeast:List of 2
    ##   .. .. .. .. ..$ lat: num 1.29
    ##   .. .. .. .. ..$ lng: num 104
    ##   .. .. .. ..$ southwest:List of 2
    ##   .. .. .. .. ..$ lat: num 1.28
    ##   .. .. .. .. ..$ lng: num 104
    ##   .. ..$ place_id          : chr "ChIJ_WZSrgsZ2jERm6wZgPqIiw0"
    ##   .. ..$ plus_code         :List of 2
    ##   .. .. ..$ compound_code: chr "7RPX+G8 Singapore"
    ##   .. .. ..$ global_code  : chr "6PH57RPX+G8"
    ##   .. ..$ types             :List of 1
    ##   .. .. ..$ : chr "street_address"
    ##  $ status : chr "OK"

For my project, I need to extract the country long and short names. The
challenge with it is that the number of elements is not static and
varies from one address to the next, which means that the position of
the country element changes too.

We can handle this problem by using the pluck() and unnest\_wider()
functions from purrr and tidyr packages, respectively. They will help us
put the data in a structured data frame object. The str\_detect()
function from the stringr package will help us get the index of the row
containing the country detail.

First, we extract the list elements we need with the pluck() function.
In my case, I need to extract address\_components. The function requires
you to specify the element’s position within the list. The
address\_components element is positioned at the top of the 3rd sublist
of the address\_all list. To extract the information from it, we will
need to pass the indexes 1, 1, 1 to the function, as shown below.

``` r
address_components <- pluck(address_all, 1, 1, 1)
address_components
```

    ## [[1]]
    ## [[1]]$long_name
    ## [1] "18"
    ## 
    ## [[1]]$short_name
    ## [1] "18"
    ## 
    ## [[1]]$types
    ## [[1]]$types[[1]]
    ## [1] "street_number"
    ## 
    ## 
    ## 
    ## [[2]]
    ## [[2]]$long_name
    ## [1] "North Canal Road"
    ## 
    ## [[2]]$short_name
    ## [1] "N Canal Rd"
    ## 
    ## [[2]]$types
    ## [[2]]$types[[1]]
    ## [1] "route"
    ## 
    ## 
    ## 
    ## [[3]]
    ## [[3]]$long_name
    ## [1] "Singapore River"
    ## 
    ## [[3]]$short_name
    ## [1] "Singapore River"
    ## 
    ## [[3]]$types
    ## [[3]]$types[[1]]
    ## [1] "neighborhood"
    ## 
    ## [[3]]$types[[2]]
    ## [1] "political"
    ## 
    ## 
    ## 
    ## [[4]]
    ## [[4]]$long_name
    ## [1] "Singapore"
    ## 
    ## [[4]]$short_name
    ## [1] "Singapore"
    ## 
    ## [[4]]$types
    ## [[4]]$types[[1]]
    ## [1] "locality"
    ## 
    ## [[4]]$types[[2]]
    ## [1] "political"
    ## 
    ## 
    ## 
    ## [[5]]
    ## [[5]]$long_name
    ## [1] "Singapore"
    ## 
    ## [[5]]$short_name
    ## [1] "SG"
    ## 
    ## [[5]]$types
    ## [[5]]$types[[1]]
    ## [1] "country"
    ## 
    ## [[5]]$types[[2]]
    ## [1] "political"
    ## 
    ## 
    ## 
    ## [[6]]
    ## [[6]]$long_name
    ## [1] "048830"
    ## 
    ## [[6]]$short_name
    ## [1] "048830"
    ## 
    ## [[6]]$types
    ## [[6]]$types[[1]]
    ## [1] "postal_code"

Next, we will use the unnest\_wider() function in order to convert the
data format from its current list type to a data frame. All unnest
functions from the tidyr package work only with the data frame type
objects, so, first, we need to put our list into a data frame format. We
do it with the tibble() function, as shown below.

``` r
address_components_df <- tibble(cols = address_components)
address_components_df
```

    ## # A tibble: 6 x 1
    ##   cols            
    ##   <list>          
    ## 1 <named list [3]>
    ## 2 <named list [3]>
    ## 3 <named list [3]>
    ## 4 <named list [3]>
    ## 5 <named list [3]>
    ## 6 <named list [3]>

You can see that the resulting data frame consistst of 6 rows, each
containing a list of 3 elements.

The unnest\_wider() function will help us turn each element of the list
into a column. The function requires to specify a vector with the names
of the list elements that we want to convert, which we can quickly
achieve with the names() function.

``` r
cols <- names(address_components_df$cols[[1]])
cols
```

    ## [1] "long_name"  "short_name" "types"

Now, let’s use the unnest\_wider() function on our data frame to see how
it works.

``` r
address_components_df %>% unnest_wider(cols) 
```

    ## # A tibble: 6 x 3
    ##   long_name        short_name      types     
    ##   <chr>            <chr>           <list>    
    ## 1 18               18              <list [1]>
    ## 2 North Canal Road N Canal Rd      <list [1]>
    ## 3 Singapore River  Singapore River <list [2]>
    ## 4 Singapore        Singapore       <list [2]>
    ## 5 Singapore        SG              <list [2]>
    ## 6 048830           048830          <list [1]>

You notice that the types column still contains the list-type objects
because it has an additional nested element which was not taken care of
by the first unnest function. To extract this data, we need to apply the
same function one more time, but now only to the types column.

``` r
address_components_df <- address_components_df %>% unnest_wider(cols) %>% unnest_wider("types") 

address_components_df <- address_components_df %>% 
  unite("Type", 3:ncol(address_components_df), sep=" ", na.rm = TRUE) #we need to concatenate all the possible types fields to facilitate string matching
```

| long\_name       | short\_name     | Type                   |
|:-----------------|:----------------|:-----------------------|
| 18               | 18              | street\_number         |
| North Canal Road | N Canal Rd      | route                  |
| Singapore River  | Singapore River | neighborhood political |
| Singapore        | Singapore       | locality political     |
| Singapore        | SG              | country political      |
| 048830           | 048830          | postal\_code           |

Now, all we need to do is to find the index of the row containing the
country detail which is accomplished with str\_detect() function.

``` r
row_index <- which(str_detect(address_components_df$Type, "country")==TRUE)
row_index
```

    ## [1] 5

Great! Now we know the row that we need to extract to get the country
detail. Let’s see what it gets us.

``` r
address_components_df[row_index, 1:2]
```

    ## # A tibble: 1 x 2
    ##   long_name short_name
    ##   <chr>     <chr>     
    ## 1 Singapore SG

Exactly what we needed!

#### Step 6 - Adding efficiency with a for loop

To make this process more efficient we can create a for loop that will
go through all the steps above for every single address in our list and
save us loads of time.

``` r
org_country <- data.frame("long_name"=NA, "short_name" = NA) #creating a shell

for (i in 1:length(org_address)){
  address <- geocode(org_address[i], output="all")
  
  if(!is.data.frame(address)){ #only proceed if the call was successful
    
    address_components <- pluck(address, 1, 1, 1)
    address_components_df <- tibble(cols = address_components)
    
    address_components_df <- address_components_df %>% 
      unnest_wider(cols) %>% unnest_wider("types") 
    
    address_components_df <- address_components_df %>% 
      unite("Type", 3:ncol(address_components_df), sep=" ", na.rm = TRUE)
  
    row_index <- which(str_detect(address_components_df$Type, "country")==TRUE)
    
    org_country[i, ] <- address_components_df[row_index, 1:2]
  }
}

knitr::kable(org_country)
```

| long\_name     | short\_name |
|:---------------|:------------|
| Singapore      | SG          |
| Colombia       | CO          |
| Russia         | RU          |
| United Kingdom | GB          |
| Hong Kong      | HK          |

Voila! Doesn’t it look great?

#### Step 7 - Tying it back together

The remaining step is to add the org\_country to our original addresses
data, and we are all set.

``` r
addresses <- cbind(addresses, org_country) 

knitr::kable(addresses)
```

| org\_address                                                       | long\_name     | short\_name |
|:-------------------------------------------------------------------|:---------------|:------------|
| 18N CanalRd Singapore 48830                                        | Singapore      | SG          |
| Cra. 13 8525 BogotaColombia                                        | Colombia       | CO          |
| RUSSIA 109004 G MOSKVA UL DOBROVOLCHESKAYA DOM 12                  | Russia         | RU          |
| 127 Ledbury Rd, Notting H ill, London W11 2AQ                      | United Kingdom | GB          |
| Shop 4C-D Tower 1 PL/F, China HK City 33 Canton road Tsim Sha Tsui | Hong Kong      | HK          |

In my project, I had about 2,000 unique originator addresses and 2,500
unique beneficiary addresses. Fortunately, Google was able to recognize
most of them correctly, and I only had to manually review about a
hundred observations, which is not bad at all.

I was able to identify several inconsistencies where the country codes
generated from parsing messages by the bank’s transaction processing
system were inaccurate. Luckily, most of them were caught and corrected
after the fact thanks to proper internal controls. But the risk remains
until these messages adhere to a better-structured format using explicit
delimiters and standardized address formats.

Back in 2017, SWIFT announced that it would remove free-format message
fields to ensure that payer and beneficiary data is systematically
captured and exchanged. This change is expected to take effect in
November 2020. In the meantime, the banks need to be vigilant in
ensuring the quality of their data.
