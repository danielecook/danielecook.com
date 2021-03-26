---
title: "A Strava plugin for bitbar"
date: 2019-06-25T20:26:29+01:00
draft: true
tags:
  - strava
  - cycling
---

[Bitbar](https://getbitbar.com/) is a cool little app for MacOS that allows you to run tiny programs that display useful information in the menu bar. These can be anything from current weather conditions, to information about digital ocean droplets, or a timer application. Being that I am also a big fan of strava, I decided to put together a strava plugin for bitbar.

![bitbar strava](/bitbar.png)
<br /><small>Wish I could put up some more impressive numbers here!</small>

The plugin displays:

* Total miles run for all time
* Total miles biked for all time
* Breakdowns of miles run/biked this year, and in the past four weeks.

To use the strava bitbar plugin:

1. You can download the binary [here](/strava.15m.cgo). Add it to your bitbar folder.
1. Go to the [Strava API](https://www.strava.com/settings/api), and create a new application.
1. Copy the `access_code`. Select `Set Access Token` from the menu.
1. Hit `⌘ + R` and you should see your stats!

If you would like to review / compile the code yourself, the source is available below or from [this gist](https://gist.github.com/danielecook/4213ec83b2af75bbe9d45f3ad03ca48d).

{{< highlight go "hl_lines=5-6,linenos=table" >}}
package main

/*
# <bitbar.title>Strava</bitbar.title>
# <bitbar.version>v0.0.1</bitbar.version>
# <bitbar.author>Daniel Cook</bitbar.author>
# <bitbar.author.github>danielecook</bitbar.author.github>
# <bitbar.desc>Bitbar Strava tracker</bitbar.desc>
*/

import (
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"os"
	"os/exec"
	"strings"
	"time"

	keychain "github.com/keybase/go-keychain"
	"github.com/tidwall/gjson"
)

const service = "Strava-API-BitBar"
const account = "access_token"
const access_group = "strava.api.bitbar"

var client = &http.Client{Timeout: 10 * time.Second}

type athlete struct {
	Bar string
}

func main() {

	args := os.Args

	/*
	   Set Access Token
	*/
	if len(args) > 1 && args[1] == "set_api_key" {
		fmt.Printf("---\n")
		cmd := exec.Command("osascript", "-e", "display dialog \"Enter Strava-API access_token\" default answer \" \" buttons {\"ok\"} with title \"Enter Strava access_token\" \n set a to the text returned of the result\n")
		out, err := cmd.CombinedOutput()
		out = []byte(strings.Trim(string(out), " \n"))
		store_api_key(out)
		if err != nil {
			log.Fatal(err)
		}

		fmt.Printf("%s\n", out)
		os.Exit(0)
	}

	var access_token = fetch_api_key()
	if access_token == "" {
		fmt.Println("Set Strava Access Token")
		fmt.Println("---")
		access_token_line()
		os.Exit(0)
	}

	// Fetch urls
	var athlete_url = fmt.Sprintf("https://www.strava.com/api/v3/athlete?access_token=%s", access_token)
	athlete := getUrl(athlete_url)
	// Error in response
	var is_err = false
	gjson.Get(athlete, "errors").ForEach(func(key, value gjson.Result) bool {
		println(gjson.Get(athlete, "message").String())
		is_err = true
		return false
	})

	if is_err == false {
		user_id := gjson.Get(athlete, "id").String()

		url := fmt.Sprintf("https://www.strava.com/api/v3/athletes/%s/stats?access_token=%s", user_id, access_token)
		stats := getUrl(url)
		fmt.Printf("🏃‍♂️ %.0f ", meters_to_miles(gjson.Get(stats, "all_run_totals.distance").Float()))
		fmt.Printf("🚲 %.0f\n", meters_to_miles(gjson.Get(stats, "all_ride_totals.distance").Float()))
		fmt.Println("---")

		fmt.Printf("%s | href=https://www.strava.com/athletes/%s\n", gjson.Get(athlete, "username").String(), user_id)

		/* This Year */
		fmt.Println("Year to date")
		fmt.Printf("🏃‍♂️ %.0f miles (%v runs) | href=https://www.strava.com/athletes/%v/training/log?sport=Run \n",
			meters_to_miles(gjson.Get(stats, "ytd_run_totals.distance").Float()),
			gjson.Get(stats, "ytd_run_totals.count").Int(),
			user_id)
		fmt.Printf("🚲 %.0f miles (%v rides) | href=https://www.strava.com/athletes/%v/training/log?sport=Ride \n",
			meters_to_miles(gjson.Get(stats, "ytd_ride_totals.distance").Float()),
			gjson.Get(stats, "ytd_ride_totals.count").Int(),
			user_id)

		/* Last Four Weeks */
		fmt.Println("4 weeks")
		fmt.Printf("🏃‍♂️ %.0f miles (%v runs) | href=https://www.strava.com/athletes/%v/training/log?sport=Run \n",
			meters_to_miles(gjson.Get(stats, "recent_run_totals.distance").Float()),
			gjson.Get(stats, "recent_run_totals.count").Int(),
			user_id)
		fmt.Printf("🚲 %.0f miles (%v rides) | href=https://www.strava.com/athletes/%v/training/log?sport=Ride \n",
			meters_to_miles(gjson.Get(stats, "recent_ride_totals.distance").Float()),
			gjson.Get(stats, "recent_ride_totals.count").Int(),
			user_id)

		/* Access Token */
		fmt.Println("---")
	}
	access_token_line()
	os.Exit(0)

}

func getUrl(url string) string {
	resp, err := client.Get(url)
	if err != nil {
		fmt.Println("API Error")
		fmt.Println("---")
		access_token_line()
		os.Exit(0)
	}
	b, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println("IO Error")
		fmt.Println("---")
		access_token_line()
		os.Exit(0)
	}
	return string(b)
}

func store_api_key(api_key []byte) {
	/*
	   Stores the API key
	*/
	item := keychain.NewItem()
	item.SetSecClass(keychain.SecClassGenericPassword)
	item.SetService(service)
	item.SetAccount(account)
	item.SetLabel("Strava API Key for BitBar")
	item.SetAccessGroup(access_group)
	item.SetData(api_key)
	item.SetSynchronizable(keychain.SynchronizableNo)
	item.SetAccessible(keychain.AccessibleWhenUnlocked)
	err := keychain.AddItem(item)
	if err == keychain.ErrorDuplicateItem {
		// If duplicate -- overwrite
		delete_api_key()
		keychain.AddItem(item)
	} else {
		log.Fatal(err)
	}
}

func delete_api_key() {
	/*
	   Deletes the record for the API key
	*/
	item := keychain.NewItem()
	item.SetSecClass(keychain.SecClassGenericPassword)
	item.SetService(service)
	item.SetAccount(account)
	keychain.DeleteItem(item)
}

func fetch_api_key() string {
	/*
	   Fetches the user API key from the keychain
	*/
	var password string
	query := keychain.NewItem()
	query.SetSecClass(keychain.SecClassGenericPassword)
	query.SetService(service)
	query.SetAccount(account)
	query.SetAccessGroup(access_group)
	query.SetMatchLimit(keychain.MatchLimitOne)
	query.SetReturnData(true)
	results, err := keychain.QueryItem(query)
	if err != nil {
		// Error
	} else if len(results) != 1 {
		// Not found
	} else {
		password = string(results[0].Data)
	}
	return password
}

func access_token_line() {
	args := os.Args
	fmt.Println("Set Access Token | bash=" + args[0] + " param1=set_api_key terminal=false refresh=true")
}

func meters_to_miles(meters float64) float64 {
	return meters * 0.0006213712
}

{{< /highlight >}}