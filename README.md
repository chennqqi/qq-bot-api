# Golang bindings for the Coolq HTTP API

[![GoDoc](https://godoc.org/github.com/catsworld/qq-bot-api?status.svg)](https://godoc.org/github.com/catsworld/qq-bot-api)
[![Build Status](https://travis-ci.org/catsworld/qq-bot-api.svg?branch=master)](https://travis-ci.org/catsworld/qq-bot-api)

If you're familiar with [go-telegram-bot-api](https://github.com/go-telegram-bot-api/telegram-bot-api),
you should quickly get familiar with this. -- Method names and architectures are just like the Telegram Bot API.

Currently it supports HTTP POST as event report method only.

Head through the following examples and [godoc](https://godoc.org/github.com/catsworld/qq-bot-api) will give you a tutorial about how to use this package.
If you still have problems, look up to the code or open an issue.

## Examples

This is a very simple bot that just displays any gotten updates, then replies it to that chat.

```go
func main() {
	bot, err := qqbotapi.NewBotAPI("MyCoolqHttpToken", "http://localhost:5700", "CQHTTP_SECRET")
	if err != nil {
		log.Fatal(err)
	}

	bot.Debug = true

	u := qqbotapi.NewWebhook("/webhook_endpoint")
	u.PreloadUserInfo = true
	updates := bot.ListenForWebhook(u)
	go http.ListenAndServe("0.0.0.0:8443", nil)

	for update := range updates {
		if update.Message == nil {
			continue
		}

		log.Printf("[%s] %s", update.Message.From.String(), update.Message.Text)

		msg := qqbotapi.NewMessage(update.Message.Chat.ID, update.Message.Chat.Type, update.Message.Text)
		bot.Send(msg)
	}
}
```

If you need to utilize a sync response, you may use a slightly different method.

```go
func main() {
	bot, err := qqbotapi.NewBotAPI("MyCoolqHttpToken", "http://localhost:5700", "CQHTTP_SECRET")
	if err != nil {
		log.Fatal(err)
	}

	bot.Debug = true

	u := qqbotapi.NewWebhook("/webhook_endpoint")
	u.PreloadUserInfo = true
	bot.ListenForWebhookSync(u, func(update qqbotapi.Update) interface{} {

		log.Printf("[%s] %s", update.Message.From.String(), update.Message.Text)

		return map[string]interface{}{
			"reply": update.Message.Text,
		}
	})

	http.ListenAndServe("0.0.0.0:8443", nil)
}
```

Update.Message.Message is a group of Media, defined in package cqcode.

```go
	for update := range updates {
		if update.Message == nil {
			continue
		}

		for _, media := range *update.Message.Message {
			switch m := media.(type) {
			case *cqcode.Image:
				fmt.Printf("The message includes an image, id: %s, url: %s", m.FileID, m.URL)
			}
		}
	}
```

There are also some useful command helpers.

```go
	for update := range updates {
		if update.Message == nil {
			continue
		}

		// If this is true, a valid command must start with "/", false by default.
		cqcode.StrictCommand = true

		if update.Message.IsCommand() {
			// cmd string, args []string
			// In a StrictCommand mode, the beginning "/" will be stripped off.
			cmd, args := update.Message.Command()

			// Note that cmd and args is still media
			cmdMedia, _ := cqcode.ParseMessage(cmd)
			for _, v := range cmdMedia {
				switch v.(type) {
				case *cqcode.At:
					fmt.Print("The command includes an At!")
				case *cqcode.Face:
					fmt.Print("The command includes a Face!")
				}
			}
		}
	}
```

Media types defined in package cqcode can be sent directly.

```go
	// Send a text message
	bot.SendMessage(10000000, "group", cqcode.Text{
		Text: "[<- These will be encoded ->]",
	})

	// Send a location
	bot.SendMessage(10000000, "group", cqcode.Location{
		Content:   "上海市徐汇区交通大学华山路1954号",
		Latitude:  31.198878,
		Longitude: 121.436381,
		Style:     1,
		Title:     "位置分享",
	})
```

You can also send a message that contains a number of media.

```go
	message := make(cqcode.Message, 0)
	message.Append(&cqcode.At{QQ: "all"})
	message.Append(&cqcode.Text{Text:" 大家起来嗨"})
	face, _ := cqcode.NewFaceFromName("调皮")
	message.Append(face)
	bot.SendMessage(10000000, "group", message)
```

In addition to manually declaring media variables,
there are some helpers that help you format images or records.

```go
	// Format a base64-encoded image (Recommended)
	image1, err := qqbotapi.NewImageBase64("/path/to/image.jpg")

	// Format an image in a file://path/to/image.jpg format. You should only use this when CQ HTTP and your bot are in the same host.
	image2 := qqbotapi.NewImageLocal("/path/to/image.jpg")

	// Format an image in the web.
	u, err := url.Parse("https://img.rikako.moe/i/D1D.jpg")
	image3 := qqbotapi.NewImageWeb(u)
	image3.DisableCache()
```

SendMessage returns a message with a message id.

```go
	m, err := bot.SendMessage(10000000, "group", "aaaaaa")
	if err == nil {
		bot.DeleteMessage(m.MessageID)
	}
```

Instead of using shortcuts like bot.SendMessage,
you can manually use the function bot.Send and bot.Do with a config.
You should find this quite familiar if you have once developed a Telegram bot.
```go
	// Alternaive to the code above
	message := qqbotapi.NewMessage(10000000, "group", "aaaaaa")
	m, err := bot.Send(message)
	if err == nil {
		config := qqbotapi.DeleteMessageConfig{MessageID: m.MessageID}
		bot.Do(config)
	}
```