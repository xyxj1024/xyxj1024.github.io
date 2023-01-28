---
layout:             post
title:              "Python Weather Data Scraping"
category:           "Data Structures, Algorithms, Programming Languages"
tags:               web-scraping python
permalink:          /posts/web-scraping/weather
last_modified_at:   "2023-01-28"
---

It was a cozy Midwest Sunday morning. I woke up a little later than usual and just couldn't decide whether I should go to the lab. Typically, to deal with this situation, I will first check the weather using iPhone's built-in weather app. However, a random thought came to me: how about letting the Terminal display weather information for me?

<!-- excerpt-end -->

Initially, my idea was to build a simple interactive Python program that takes a string (the city name, e.g., "Saint Louis") from standard input and returns relevant weather information to the standard output. By including it in the <code>.bash_profile</code> file, whenever I open the Terminal, the Bash shell will run this program for me. I did some research and found that the popular approach is to pull certain data from web pages using <code>BeautifulSoup</code> library with an HTML parser. A HTTP request needs a proper header. There are three fields we can put into the header: user agent, accept language, and content language:

```python
# Go find the latest user agent for your browser
USER_AGENT = "Mozilla/5.0 (Macintosh; Intel Mac OS X 12_6) \
              AppleWebKit/537.36 (KHTML, like Gecko) \
              Chrome/106.0.0.0 Safari/537.36"
LANGUAGE = "en-US,en;q=0.5"
headers = {
    'User-Agent': USER_AGENT, 
    'Accept-Language': LANGUAGE, 
    'Content-Language': LANGUAGE
}
```

What I need is a function that filters HTML information and converts it into different output strings. I wrote a <code>weather()</code> function that looked like this:

```python
def weather(city):
    city_weather = city + " weather"
    city_weather = city_weather.replace(" ", "+")
    res = requests.get(f'https://www.google.com/search?q={city_weather}&oq={city_weather}&aqs=chrome.0.69i59j0i20i263i512j0i512l8.3487j1j9&sourceid=chrome&ie=UTF-8', headers = headers)
    # print("Searching......\n")
    # create a new soup
    soup = BeautifulSoup(res.text, "html.parser")
    # extract region
    location = soup.select('#wob_loc')[0].getText().strip()
    # extract the day and hour now
    time = soup.select('#wob_dts')[0].getText().strip()
    # get the actual weather
    weather = soup.select('#wob_dc')[0].getText().strip()
    # extract temperature now
    temperature = soup.select('#wob_tm')[0].getText().strip()
    # fahrenheit to celsius with floating point truncation
    temperature = int((int(temperature) - 32) * 5.0 / 9.0)
    # get the precipitation
    precipitation = soup.select('#wob_pp')[0].getText().strip()
    # get the % of humidity
    humidity = soup.select('#wob_hm')[0].getText().strip()
    # extract the wind level
    wind = soup.select('#wob_ws')[0].getText().strip()
    print("temperature:   " + str(temperature) + "\u00B0C")
    print("precipitation: " + precipitation)
    print("humidity:      " + humidity)
    print("wind:          " + wind)
```

Although this program did produce desired output, after a couple of weeks, I realized that Google is not an ideal target for web scraping. For a new version of my weather data scraping program, I decided to make it visit the [National Weather Service](https://www.weather.gov/) website. Different from Google search, the URL of this website is filled with latitude and longtitude values for weather lookup. A convenient way to get these values is to inquire [ipinfo.io](ipinfo.io), which includes information pertinent to our current IP address:

```python
location = requests.get('https://ipinfo.io/loc').content.decode().strip()
latitude = location.split(',')[0]
longtitude = location.split(',')[1]
```

Out of curiosity, we can also record:

```python
city = requests.get('https://ipinfo.io/city').content.decode().strip()
region = requests.get('https://ipinfo.io/region').content.decode().strip()
country = requests.get('https://ipinfo.io/country').content.decode().strip()
```

This is my new <code>weather()</code> function:

```python
def weather(lat, lon):
    res = requests.get(f'https://forecast.weather.gov/MapClick.php?\
                         lat={lat}&lon={lon}', headers = headers)
    soup = BeautifulSoup(res.text, "html.parser")
    items = soup.find_all("div", class_ = "tombstone-container")
    times = [item.find(class_ = "period-name").get_text(separator = " ") for item in items]
    weathers = [item.find(class_ = "short-desc").get_text(separator = " ") for item in items]
    temp = re.compile('.*temp.*')
    temperatures = [item.find(class_ = temp).get_text(separator = " ") for item in items]
    print("".join(f'{times[i]:^24}' for i in range(4)))
    print("".join(f'{weathers[i]:^24}' for i in range(4)))
    print("".join(f'{temperatures[i]:^24}' for i in range(4)))
    print("\n")
```

Finally, I got my daily weather checker working:

```console
Last login: Sun Sep 18 13:29:32 on ttys000

Another beautiful day in Chesterfield, Missouri, US:

      Tonight            Wednesday        Wednesday Night         Thursday      
       Clear               Sunny           Partly Cloudy           Sunny        
     Low: 24 °F         High: 51 °F          Low: 33 °F         High: 68 °F
```