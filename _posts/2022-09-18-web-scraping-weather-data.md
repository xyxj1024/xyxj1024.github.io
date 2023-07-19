---
layout:             post
title:              "Python Weather Data Scraping"
category:           "Programming Languages"
tags:               web-scraping python
permalink:          /blog/web-scraping/weather-data
last_modified_at:   "2023-01-30"
---

It was a cozy Midwest Sunday morning. I woke up a little later than usual and just couldn't decide whether I should go to the lab. Typically, to deal with this situation, I will first check the weather using iOS's built-in weather app. However, a random thought came to me: why not just let the Terminal display weather information for me?

<!-- excerpt-end -->

Initially, my idea was to build a simple interactive Python program that takes a string (e.g., a city name like "Saint Louis") from standard input and returns the corresponding weather information to the standard output. By including it in the <code>.bash_profile</code> file, whenever I opened the Terminal, the Bash shell would automatically run this program for me. I did some research and found that one popular approach is to pull certain data from web pages using <code>BeautifulSoup</code> library with an HTML parser. A HTTP request needs a proper header. There are three fields we can put into the header: user agent, accept language, and content language:

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

Although this program did produce desired output, after a couple of weeks, I realized that Google is not an ideal target for web scraping. For a new version of my weather data scraping program, I asked it to visit the [National Weather Service Website](https://www.weather.gov/) instead. Different from Google search, the URL of this website is filled with latitude and longtitude values for weather lookup. A convenient way to obtain these values is to inquire [ipinfo.io](ipinfo.io), which stores various information about our current IP address:

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
    res = requests.get(f'https://forecast.weather.gov/MapClick.php?lat={lat}&lon={lon}', headers = headers)
    soup = BeautifulSoup(res.text, "html.parser")

    times = soup.find_all("p", {"class": "period-name"})
    descs = soup.find_all("p", {"class": "short-desc"})
    temps = soup.find_all("p", {"class": re.compile('.*temp.*')})
    times = [time.get_text(separator = " ") for time in times]
    descs = [desc.get_text(separator = " ") for desc in descs]
    temps = [temp.get_text(separator = " ") for temp in temps]
    # Convert Fahrenheit into Celsius
    for i, temp in enumerate(temps):
        temp = re.sub(r'\D', '', temp)
        temp = int((int(temp) - 32) * (5 / 9))
        temp = str(str(temp) + " \u00B0C")
        temps[i] = temp
    # Shorten weather descriptions
    for j, desc in enumerate(descs):
        new_desc = ""
        for k, char in enumerate(desc):
            if k > 20 and char == " ":
                new_desc += "..."
                break
            new_desc += char
        descs[j] = new_desc
    # Print results
    print("".join(f'{times[i]:^30}' for i in range(4)))
    print("".join(f'{descs[i]:^30}' for i in range(4)))
    print("".join(f'{temps[i]:^30}' for i in range(4)))
    print("\n")
```

Finally, I got my daily weather checker working:

```console
Last login: Mon Feb  6 12:14:17 on ttys000
Another beautiful day in St. Louis, Missouri, US:

大雨落幽燕，白浪滔天，秦皇岛外打鱼船。一片汪洋都不见，知向谁边？
往事越千年，魏武挥鞭，东临碣石有遗篇。萧瑟秋风今又是，换了人间。

        This Afternoon                   Tonight                       Tuesday                    Tuesday Night         
   Mostly Cloudy and Breezy     Cloudy and Breezy then...      Slight Chance Showers...            Chance Rain          
            15 °C                          8 °C                          9 °C                          3 °C             
```