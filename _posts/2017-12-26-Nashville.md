---
title: "Open Street Maps - Nashville"
date: 2017-12-26
tags: [data science, python]
header:
  image: "/images/Nashville/Nashville.jpg"
excerpt: "Data Science, Nashville"
mathjax: "true"
---

# Open Street Maps - Nashville

### Index:

1. Map Area
2. Problems
3. Data Overview
4. User Overview
5. Top 10 - cities by count
6. Top 10 amenities
7. Top 10 cuisines
8. Top 10 sources of data
9. Top 10 natural areas
10. Additional Ideas
11. Sources

### Map Area

Nashville, TN, United States

https://mapzen.com/data/metro-extracts/metro/nashville_tennessee/

I have been in Nashville for 5 years and I am curious to see what database querying reveals.

### Problems:

1. A lot of street names are not identified by the right key: "street", rather are just marked with the key: "name"
2. Streets types are abbreviated ("E Main St" instead of "E Main Street")
3. Some of the streets are only listed in three seperate segments: name_direction_prefix, name_base, name_type (W, End, Ave)
4. Some postcodes have the state name abbreviated (TN 30307)
5. Some postcodes are longer than 5 digits (37207-4405)
6. Some house numbers have street names as well (417 Woodland St)
7. Some city names have the state or county name in them ("Gallatin, TN", "Nashville-Davidson")

#### Problem 1: Street names not identified by the key "street"

Since most streets were under the key:"name", I had to find a way to isolate the street names from other names.
1. I created a list of all possible street types appearing in the data set under the list "streets"
2. If the name contains the street type anywhere in the string, it is considered for the next step
3. Next step is to check if the last word is a string type (Washington St.) or the street direction (Washingtion St. N)
4. If yes, then I assign this tag the key:"street" and type:"addr"


```python
streets = ["Street", "St", "St.", "Avenue", "Ave", "Boulevard", "Blvd", "Drive", "Dr", "Court", "Ct", "Place",
           "Lane", "Ln", "Road", "Rd", "Run", "spur", "Path", "Trail", "Parkway", "Pky", "Pkwy", "Commons", "Pike",
           "Pik", "Alley", "Pl", "Way", "Terrace", "Circle", "Row", "Cv", "Tcre", "Loop", "Hwy", "Br", "Xing", "Plz",
           "Byp", "Pass", "Walk", "Cres", "Ter"]
street_direction = ["North", "N", "South", "S", "East", "E", "West", "W"]

street_last = re.compile(r'\S+\.?$', re.IGNORECASE)
streets_re = re.compile(r'\b(?:%s)\b' % '|'.join(streets))


if way_tags_dict['key'] == 'name':
                    if streets_re.search(way_tags_dict['value']):   # Check the value contains one of the street type
                        m = street_last.search(way_tags_dict['value'])
                        if m.group() in streets or m.group() in street_direction:# Check if the last word is a street type
                            way_tags_dict['key'] = 'street'
                            way_tags_dict['type'] = 'addr'
```

#### Problem 2: Street types are abbreviated

1. Created a mapping of abbreviated street types and the full street type
2. Created a function update_name to replace the abbreviated street type with the full street type


```python
mapping = { "St": "Street",
            "St.": "Street",
            "Ave": "Avenue",
            "Rd.": "Road",
            "Rd": "Road",
            "Pky": "Parkway",
            "Pkwy": "Parkway",
            "Pl": "Place",
            "Dr": "Drive",
            "Ct": "Court",
            "Pik": "Pike",
            "Blvd": "Boulevard",
            "Cv": "Cove",
            "Trce": "Trace",
            "Hwy": "Highway",
            "Br": "Branch",
            "Ln": "Lane",
            "Xing": "Crossing",
            "Plz": "Plaza",
            "Byp": "Bypass",
            "Cres": "Crescent",
            "Ter": "Terrace"
            }


def update_name(name):

    m = street_last.search(name)
    if m.group() in mapping.keys():
        name = name.replace(m.group(),mapping[m.group()])

    return name
```

#### Problem 3: Street listed only in three seperate segments

These seperate segments were only needed if the street name was not found in a tag
1. If the whole street name was found in a tag, I created a flag to indicate that the street name is found
2. If the street_found flag is set, no need to look at the segments name_base, name_type, name_direction_prefix
3. If the whole street name is not available, look up to see if the name_type is in streets
4. If yes, create a new street name by contatenating the name_direction_prefix, name_base, name_type


```python
        if street_found:
            pass
        else:           # If there are any streets which are only available in the key:name_base/name_type
            if street_base: # If street base name is available
                if street_type in streets: # If street type is one of the valid street types
                    way_tags_dict['id'] = element.attrib['id']
                    way_tags_dict['key'] = 'street'
                    way_tags_dict['type'] = 'addr'

                    if street_dir:                                      # If street direction is available
                        way_tags_dict['value'] = street_dir + ' ' + street_base + ' ' + street_type
                    else:
                        way_tags_dict['value'] = street_base + ' ' + street_type

                    way_tags_dict['value'] = update_name(way_tags_dict['value'])
```

#### Problem 4/5/6: Postal codes with state names and House numbers with street names

1. The postal code and house number should not contain any character other than number or "-"
2. "-" needs to be allowed for postal codes like 37076-8885 and for house numbers like 401-409
3. Wrote a regex function to do this


```python
            if way_tags_dict['key'] in ('postcode','housenumber'):
                # Extract number-number and exclude other characters (like state = 'TN 30307' or housenumber = '417 Woodland St')
                if re.findall(r'\d+-\d+|\d+',way_tags_dict['value']):
                    way_tags_dict['value'] = re.findall(r'\d+-\d+|\d+',way_tags_dict['value'])[0]
```

#### Problem 7: City names with the state name or the county name

1. The city name should not contain the state name (Gallatin, TN) or the county name (Nashville-Davidson)
2. Wrote a regex function to split the string using alphabets and space ['Gallatin', 'TN']
3. Picket the first part of the list as the city name ('Gallatin')
4. Used an or condition in the regex to first look for cities with more than one word, and then look for cities with one word


```python
            if way_tags_dict['key'] == 'city':
                if re.findall(r'\w+\s\w+|\w+', way_tags_dict['value']):
                    way_tags_dict['value'] = re.findall(r'\w+\s\w+|\w+', way_tags_dict['value'])[0]
```

### Data Overview

This section contains basic statistics about the dataset

#### File Size

1. nashville_tennessee.osm..........290 MB
2. openstreetmaps.db................170 MB
3. nodes.csv..............................109 MB
4. nodes_tags.csv.........................4.89 MB
5. ways.csv...............................8.17 MB
6. ways_nodes.csv.........................36.12 MB
7. ways_tags.csv...........................26.8 MB

#### Number of nodes

sqlite> SELECT COUNT(*) FROM nodes;
1360861

#### Number of ways

sqlite> SELECT COUNT(*) FROM ways;
141343

### User Overview

#### Top 10 contributing users


```python
SELECT A.user,COUNT(*) as num
FROM (SELECT user FROM nodes UNION ALL SELECT user FROM ways) A
GROUP BY A.user
ORDER BY num DESC
LIMIT 10;

woodpeck_fixbot,273147
"Shawn Noble",178065
st1974,96254
AndrewSnow,57234
Rub21,53110
TIGERcnl,52374
StevenTN,29495
maxerickson,28553
darksurge,27541
```

Total Contribution by top 10 contributors:


```python
select sum(num)
from (SELECT A.user,COUNT(*) as num
FROM (SELECT user FROM nodes UNION ALL SELECT user FROM ways) A
GROUP BY A.user
ORDER BY num DESC
LIMIT 10) B;

822379
```

#### Total Contributors


```python
SELECT count(distinct A.user) as num
     FROM (SELECT user FROM nodes UNION ALL SELECT user FROM ways) A;

1101
```

Total contributions:


```python
SELECT count(*) as num
     FROM (SELECT user FROM nodes UNION ALL SELECT user FROM ways) A;

1502204
```

#### One time contributors


```python
SELECT COUNT(*)
FROM
    (SELECT A.user, COUNT(*) as num
     FROM (SELECT user FROM nodes UNION ALL SELECT user FROM ways) A
     GROUP BY A.user
     HAVING num=1)  B;

190
```

### User Summary:

1. Around 55% of the total contribution came from the top 10 users
2. Around 17% of the users are one time contributors

### Top 10 cities by count


```python
SELECT UPPER(tags.value), COUNT(*) as count
FROM (SELECT * FROM nodes_tags UNION ALL
      SELECT * FROM ways_tags) tags
WHERE UPPER(tags.key) = 'CITY'
GROUP BY UPPER(tags.value)
ORDER BY count DESC
LIMIT 10;

CLARKSVILLE,10425
FRANKLIN,559
"SPRING HILL",387
NASHVILLE,330
MURFREESBORO,269
BRENTWOOD,225
HERMITAGE,43
COLUMBIA,35
NOLENSVILLE,17
SPRINGFIELD,16
```

The metro extract for Nashville includes several surrounding cities and not just the Nashville metro

### Top 10 postal codes


```python
SELECT tags.value, COUNT(*) as count
FROM (SELECT * FROM nodes_tags
	  UNION ALL
      SELECT * FROM ways_tags) tags
WHERE tags.key='postcode'
GROUP BY tags.value
ORDER BY count DESC
LIMIT 10;

37042,9950
37064,538
37040,445
37027,239
37129,231
37211,122
37174,101
37203,76
38401,56
37076,44
```

The postal code (37042) with the highest frequency does belong to the city with the highest frequency (Clarksville)

### Top 10 amenities


```python
SELECT tags.value,count(*) as num
FROM (SELECT * FROM nodes_tags UNION ALL
      SELECT * FROM ways_tags) tags
WHERE tags.key = 'amenity'
group by tags.value
order by num desc
limit 10;

grave_yard,3483
place_of_worship,2519
parking,1703
school,1479
restaurant,394
fast_food,292
parking_space,236
fuel,185
post_office,115
bank,102
```

Surprised to learn that graveyard is the number 1 amenity!

### Top 10 cuisines


```python
SELECT tags.value,count(*) as num
FROM (SELECT * FROM nodes_tags UNION ALL
      SELECT * FROM ways_tags) tags
WHERE tags.key = 'cuisine'
group by tags.value
order by num desc
limit 10;

burger,99
mexican,65
sandwich,31
american,28
coffee_shop,26
pizza,23
chicken,19
ice_cream,14
japanese,13
regional,13
```

### Top 10 sources of data


```python
SELECT tags.value,count(*) as num
FROM (SELECT * FROM nodes_tags UNION ALL
      SELECT * FROM ways_tags) tags
where tags.key = 'source'
group by tags.value
order by num desc
limit 10;

tiger_import_dch_v0.6_20070829,15175
Bing,3655
county_import_v0.1,932
bing,555
"TIGER/Line┬« 2008 Place Shapefiles (http://www.census.gov/geo/www/tiger/)",402
"USGS Geonames",314
Yahoo,198
tiger_import_dch_v0.6_20070813,85
tiger_import_dch_v0.6_20070812,79
http://www.tdot.state.tn.us/sr840s/,64
```

In the top 10 sources we see a public and private partnership.

1. Public : (Tiger which is part of the US Dept of Commerce, tdot - tennessee dept of transportation, USGS - US Board of Geographic Names)
2. Private : Yahoo, Bing

### Top 10 natural areas


```python
SELECT tags.value,count(*) as num
FROM (SELECT * FROM nodes_tags UNION ALL
      SELECT * FROM ways_tags) tags
where tags.key = 'natural'
group by tags.value
order by num desc
limit 10;

tree,734
water,636
peak,300
wood,271
sand,89
cliff,66
tree_row,64
grassland,28
wetland,11
beach,4
```

I was curious to see "beach" in Tennessee so I decided to explore this a bit more


```python
select *
FROM (SELECT * FROM nodes_tags UNION ALL
      SELECT * FROM ways_tags) tags
where tags.id in
(SELECT tags.id
FROM (SELECT * FROM nodes_tags UNION ALL
      SELECT * FROM ways_tags) tags
where tags.value = 'beach');

56851418,ele,149,"regular
56851418,name,"Long Island Beach (historical)","regular
56851418,natural,beach,"regular
56851418,created,03/01/1990,"gnis
56851418,state_id,47,"gnis
56851418,county_id,037,"gnis
56851418,feature_id,1313463,"gnis
56851423,ele,149,"regular
56851423,name,"Willow Beach (historical)","regular
56851423,natural,beach,"regular
56851423,created,03/01/1990,"gnis
56851423,state_id,47,"gnis
56851423,county_id,037,"gnis
56851423,feature_id,1313508,"gnis
964890374,access,public,"regular
964890374,name,"Swimming Area","regular
964890374,natural,beach,"regular
964890374,surface,sand,"regular
37524225,natural,beach,"regular
37524225,surface,sand,"regular
```

Found two names: Long Island Beach and Willow Beach. Turns out that these are beaches on the Percy Priest Lake, very close to my where I live

### Additional Ideas

### Data cleaning

State names are not consistent and can be cleaned by creating a mapping of names just like street names


```python
SELECT tags.value,count(*) as num
FROM (SELECT * FROM nodes_tags UNION ALL
      SELECT * FROM ways_tags) tags
where tags.key = 'state'
group by tags.value
order by num desc
limit 50;

TN,11992
Tennessee,14
KY,8
tn,7
Tenessee,2
TB,1
Tn,1
tN,1
```

Benefit: This will improve accuracy of any analysis done by state on the openstreetmaps data. Example: finding population by state for all states. If we have consistent state names, we will have just one row for wach state.

Challenges: The current data needs to be analyzed for state names. All exceptions need to be found and entered in the mapping dictionary. The code needs to be run for the area in which clean up needs to be performed.

### Improve data collection:

If we don't want to depend on private companies to have streetmap data, it is essential to educate and incentivize people to contribute data to an open source public platform like openstreetmaps. One way to do it would be to partner with public schools and teach high school students the importance of openstreetmaps and teach them how to contribute to the openstreetmaps. It could be one of their assignments.

Benefits: This will encourage a discussion among the students and their families about the importance of publicly owned street data. Some of the students might go on to become life long contributors. A lot of additional data can be captured if this can be done in schools across states.

Challenges: It would be hard for the openstreet community to convince the public schools to include openstreetmaps contribution as an assignment in their course work. It would also be challenging to get students interested about contributing to openstreetmaps. But if done, this can really help in improving data in openstreetmaps while allowing students contribute to the community.

### Sources:

1. Types of streets: https://np.reddit.com/r/explainlikeimfive/comments/2me7l2/eli5_whats_the_difference_between_an_ave_rd_st_ln/cm3dn3j/?context=3
2. Regular expression documentation: https://docs.python.org/2/library/re.html
3. Regex Training: https://www.youtube.com/watch?v=DRR9fOXkfRE
4. Sample project: https://gist.github.com/FK85/5f0216b494bf171dc43144afcadc4d89
5. Calculate distance using lat lon: https://stackoverflow.com/questions/27928/calculate-distance-between-two-latitude-longitude-points-haversine-formula
