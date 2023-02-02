# Subway Delays and Crime in NYC

## Authors
Sara Douglas, Fritz Grunert, Mason Lonoff, Susan Lu

## Table of Contents
- [Problem Statement](#Problem Statement)
- [Requirements](#Requirements)
- [Data](#Data)
- [Data Processing](#Data_Processing)
- [Results](#Results)

## Problem Statement
	Since we live in New York City, the intersection of Subway delays and crime was interesting to us as it impacts our daily lives. After exploring the available data we decided to focus on answering the following questions:

Are NYPD complaints and subway alerts/delays related?
Does the distance to the nearest subway station affect crime?
What crimes occur most when there are subway delays?
Has the COVID-19 Pandemic had any impact on frequency of complaints or subway alerts/delays?

## Requirements
* Required python libraries for ETL and machine learning model
    - pandas
    - numpy
    - scikit-learn
    - selenium
    - beautifulsoup4
    
## Data
For our study we used three data sources.
* NYPD Complaint Data Historic | NYC Open Data (cityofnewyork.us)
This dataset includes all valid felony, misdemeanor, and violation crimes reported to the New York City Police Department (NYPD) from 2006 to the end of 2021. It also contains information about the location of the crime and the demographic information about the victim and suspect. The dataset includes years 2006-2022 with 35 columns and 7.83 million rows.
 - Data Dictionary: https://data.cityofnewyork.us/api/views/qgea-i56i/files/ee823139-888e-4ad0-badf-e18e2674a9cb?download=true&filename=NYPD_Complaint_Historic_DataDictionary.xlsx 
* NYC Transit Subway Station Map | State of New York
This data file provides a variety of information on subway station entrances and exits which includes but is not limited to: Division, Line, Station Name, Longitude and Latitude coordinates of entrances/exits.
* Alert Archive (mymtaalerts.com)
This webpage is an archive of alerts released by the MTA. It contains a date, agency, subject, and message.

## Data Processing
NYC Transit Subway Station Map:
	The list of Subway station entrances and exits had multiple locations for each station thus we reduced each station to one location and a single set of coordinates. Additionally we condensed the original 12 train line columns to one column for readability. 

Retrieval of MTA Historic Alert  Data:
	We used a tool called Selenium to automate the web-scraping of the data. We input the url to start at, and after inspecting the webpage HTML, we can pull out the buttons and forms we need to manipulate. We filter out bus alerts, elevator alerts, and anything else that is not subway data. We select start and end dates, and then the website has been prepped for scraping with BeautifulSoup4. A maximum of fifty rows were displayed per page, which resulted in ~3000 pages of data that we wanted to scrape from 2017-2021, which is why automation was necessary. We parse through the HTML using BeautifulSoup4, and retrieve each row. After the table has been fully parsed, Selenium ‘clicks’ on the next page and repeats. 
	In our initial data analysis we incorporated 2017, however after further investigation we realized 2017 data was partially incomplete and it was therefore excluded. There were a few interesting finds within each dataset. 

Retrieval of Real-Time/Recent Alerts Data Using Kafka:
	A data factory was built to automatically webscrape MTA alerts. We noticed that the alerts are assigned a code chronologically using hexadecimal. Once we determine the starting code, the code is incremented by 1 within a for loop. The loop uses beautiful soup to parse through the data to pull out alert titles, date and time, alert message, and the alert agency. Since we were only interested in subway delays data, only the alerts with an agency of ‘NYC’ were kept. We were also not interested in elevator or escalator alerts, which were included in the mix, and filtered those out using regular expression. Each alert is then sent to Kafka as a message, serving as a producer. 
	It takes time for Kafka to receive and store the messages so a two and a half minute waiting time was added. We used a consumer to pull the messages out of Kafka and append the messages to a stored data frame of the previous messages.  The data frame is then sorted so the alert codes are sorted in descending order. The entire data frame is then saved as a csv. This file is referenced in the data factory so that the next time the data factory runs, it will use the code on the first row to know which alert code it left off at.

Data Cleaning of Subway Alerts:
	After the alerts data were collected, the alerts were filtered out by update status and delay status. We did not want to consider planned delays and we noticed that those that alert titles with the affected trains are unexpected delays, as opposed to planned service delays. We also did not want alerts categorized as updates because they are referring to a previous instance, not a new one. 
Once these rows were removed, a list of the train lines was created and a new column was added to show which train line was affected. This could cause an alert to be represented multiple times since one alert can affect multiple train lines. In addition, another column was added to show what borough is affected by the alert. Finally, a Latent Dirichlet Allocation model was used to categorize the types of alerts, completing our dataset. The LDA model is a form of topic modeling. We used this unsupervised learning model to sift through the message column and decipher the underlying categories. Through a mixture of the model and domain knowledge, we were able to create six message categories. Then, we had to assign each message a category. This was done by using two keyword dictionaries that assigned a message based on a keyword(s). 

NYPD Historic Complaint Data:
	Our study focused on the years 2018-2021, reducing the row count to around 2 million. Some dates were entered incorrectly so we replaced the incorrect dates using pandas. Additionally we removed any rows lacking date of occurrence and location data. Due to the large size, the dataset had to be split up prior to processing and rejoined afterwards. In the process of uploading data, Brooklyn complaint data was deemed to be corrupted and was therefore removed from our analysis.

Joining Data Spatially and Temporally:
  Firstly we joined the complaints data with the subway station data by utilizing the coordinates provided in each dataset. For each complaint we added a column of the closest train station, the distance to that station and the train lines stopping at that station. To determine the distance between the complaint and the closest station we utilized the Haversine formula which accounts for the curvature of the earth. Each complaint is then linked temporally to delays. For each station that a delay affects, if a complaint is linked to that station within two hours after the delay, then that complaint is joined on the delay.
