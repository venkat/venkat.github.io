---
title: Twitter API authentication in Go
layout: post
category: programming
---

 This post is for developers getting started with accessing Twitter's API with Go. If you have not looked at it already, I recommend that you become familiar with how [Twitter's REST API](https://dev.twitter.com/rest/public) supports multiple [ways of authentication](https://dev.twitter.com/oauth) based on OAuth. Also, note that we will not be looking at [xAuth](https://dev.twitter.com/oauth/xauth) or [oAuth Echo](https://dev.twitter.com/oauth/echo) in this post.

To start playing with Twitter's API, you will need to [create a Twitter app](https://apps.twitter.com/). Once you do that, go to the "Keys and Access Tokens" tab in your new app. You will need the Consumer Key, Consumer Secret, Access Token, and Access Token Secret from that tab for the examples below.

Twitter's API authentication is based on OAuth 1.0a, but the [application-only authentication](https://dev.twitter.com/oauth/application-only) method is based on OAuth 2.

Go has no standard library implementation of OAuth 1.0a and [an experimental package for OAuth 2](https://github.com/golang/oauth2). However, there are at least a couple of open source implementations of OAuth 1.0a in Go. I will be using the one from [mrjones](https://github.com/mrjones/oauth). The other popular one is from [garyburd](https://github.com/garyburd/go-oauth).

## [Single-user OAuth](https://dev.twitter.com/oauth/overview/single-user)

As the developer of your application, you can use the pre-made Access Token and Access Token Secret that Twitter gives you when you create your app, so you don't have to do the three-legged OAuth dance that you need to do when you access the API on behalf of other users. [Twitter's documentation](https://dev.twitter.com/oauth/overview/single-user) provides examples of this use-case in other languages but not in Go.

Here is how you can do it in Go:

{% highlight cpp linenos %}
package main

import (
	"fmt"
	"github.com/mrjones/oauth"
	"io/ioutil"
	"log"
)

const (
	ConsumerKey       = "<YOUR CONSUMER KEY HERE>"
	ConsumerSecret    = "<YOUR CONSUMER SECRET HERE>"
	AccessToken       = "<YOUR ACCESS TOKEN HERE>"
	AccessTokenSecret = "<YOUR ACCESS TOKEN SECRET HERE>"
)

func main() {
	consumer := oauth.NewConsumer(ConsumerKey,
		ConsumerSecret,
		oauth.ServiceProvider{})
	//NOTE: remove this line or turn off Debug if you don't
	//want to see what the headers look like
	consumer.Debug(true)
	//Roll your own AccessToken struct
	accessToken := &oauth.AccessToken{Token: AccessToken,
		Secret: AccessTokenSecret}
	twitterEndPoint := "https://api.twitter.com/1.1/statuses/mentions_timeline.json"
	response, err := consumer.Get(twitterEndPoint, nil, accessToken)
	if err != nil {
		log.Fatal(err, response)
	}
	defer response.Body.Close()
	fmt.Println("Response:", response.StatusCode, response.Status)
	respBody, err := ioutil.ReadAll(response.Body)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(string(respBody))
}

{% endhighlight %}

## [Application-only oAuth](https://dev.twitter.com/oauth/application-only)

This authentication context can be used for API requests that aren't made on behalf of a specific user, like searching a tweet for example. This context is useful because of the higher rate limit that you get in addition to any rate limit for requests made on a user's behalf.

You don't need any package outside of the standard library for this type of authentication. All you need to do is implement the steps detailed in Twitter's documentation. The comments in the implementation below map to the steps mentioned in the documentation.

{% highlight cpp linenos %}
package main

import (
	"bytes"
	b64 "encoding/base64"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"net/url"
	"strconv"
)

const (
	ConsumerKey    = "<YOUR CONSUMER KEY HERE>"
	ConsumerSecret = "<YOUR CONSUMER SECRET HERE>"
)

func main() {
	client := &http.Client{}
	//Step 1: Encode consumer key and secret
	encodedKeySecret := b64.StdEncoding.EncodeToString([]byte(fmt.Sprintf("%s:%s",
		url.QueryEscape(ConsumerKey),
		url.QueryEscape(ConsumerSecret))))

	//Step 2: Obtain a bearer token
	//The body of the request must be grant_type=client_credentials
	reqBody := bytes.NewBuffer([]byte(`grant_type=client_credentials`))
	//The request must be a HTTP POST request
	req, err := http.NewRequest("POST", "https://api.twitter.com/oauth2/token", reqBody)
	if err != nil {
		log.Fatal(err, client, req)
	}
	//The request must include an Authorization header formatted as
	//Basic <base64 encoded value from step 1>.
	req.Header.Add("Authorization", fmt.Sprintf("Basic %s", encodedKeySecret))
	//The request must include a Content-Type header with
	//the value of application/x-www-form-urlencoded;charset=UTF-8.
	req.Header.Add("Content-Type", "application/x-www-form-urlencoded;charset=UTF-8")
	req.Header.Add("Content-Length", strconv.Itoa(reqBody.Len()))

	//Issue the request and get the bearer token from the JSON you get back
	resp, err := client.Do(req)
	if err != nil {
		log.Fatal(err, resp)
	}
	defer resp.Body.Close()
	respBody, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		log.Fatal(err, respBody)
	}

	type BearerToken struct {
		AccessToken string `json:"access_token"`
	}
	var b BearerToken
	json.Unmarshal(respBody, &b)

	//choose your API endpoint that supports application only auth context
	//and create a request object with that
	twitterEndPoint := "https://api.twitter.com/1.1/application/rate_limit_status.json"
	req, err = http.NewRequest("GET", twitterEndPoint, nil)
	if err != nil {
		log.Fatal(err)
	}

	//Step 3: Authenticate API requests with the bearer token
	//include an Authorization header formatted as
	//Bearer <bearer token value from step 2>
	req.Header.Add("Authorization",
		fmt.Sprintf("Bearer %s", b.AccessToken))

	//Issue the request and get the JSON API response
	resp, err = client.Do(req)
	if err != nil {
		log.Fatal(err, resp)
	}
	defer resp.Body.Close()
	respBody, err = ioutil.ReadAll(resp.Body)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("response Status:", resp.Status)
	fmt.Println("response Headers:", resp.Header)
	fmt.Println(string(respBody))
}

{% endhighlight %}

## [Three-legged Authentication](https://dev.twitter.com/oauth/3-legged)

To get started, [mrjones](https://github.com/mrjones/oauth) already provides a complete working [example code](https://github.com/mrjones/oauth/blob/master/examples/twitter/twitter.go) for Twitter's API. The README describes how you can run it, and it works great. The example code uses PIN based three-legged authentication.


I put the code in this post [on github](https://github.com/venkat/twitter_oauth).
