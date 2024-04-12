+++ 
draft = false
date = 2024-04-12T21:33:28+05:00
title = "Building A Future Trading Bot"
description = "building a future trading bot in Golang: Part 1"
slug = "ftb-1"
authors = ["Ahmed Nadeem"]
tags = ["golang", "cryptocurrency"]
categories = []
externalLink = ""
series = ["Future Trading Bot"]
+++

In this series we'll create a future trading bot in Golang, that will connect with a cryptocuurency exchange and start trading based on a strategy that we'll discuss in future blog posts. Since this is the first entry of this series, there is alot of stuff that we need to setup before we go deep into the exciting stuff - hopefully making some profit from cryptocurrenices :grin:. Moreover, I have chosen cryptocurrencies for this bot since the tech behind it is pretty solid and alot of exchnages have really good documentation for developers, but this approach could also be used for any other trading platform. Finally, I am also assuming that anyone who's reading this article is not writing their first Golang program, although Golang is pretty easy, this won't be a Golang tutorial. So let's begin.

First let's initialize the project with ```go mod init <project name>``` and let's connect it with a Postgres database. For now we'll only need the database to store api key information that we'll get from the crypto exchange. Here's how your env would look for now:

```
# Postgres Live
DB_HOST=
DB_DRIVER=postgres
DB_USER=
DB_PASSWORD=
DB_NAME=
DB_PORT=5432 #Default postgres port

#Encryption
ENCRYPTION_PASS=
```

Now let's setup our model, we'll be using Gorm to connect with the database and define our models. In models/key.go:

```Go
// models/key.go
type Key struct {
	Keyid           uuid.UUID `gorm:"primary_key;type:uuid;default:gen_random_uuid()" json:"key_id"`
	ApiKey          string    `gorm:"not null;unique" json:"api_key"`
	SecretKey       string    `gorm:"not null;unique" json:"secret_key"`
	Passphrase      string    `gorm:"" json:"passphrase"`
	UserEmail       string    `json:"user_email"`
	TradeAmount     int
	AllowedCoins    int
	CapitalPerTrade float64
	Start           bool
}
```

Trade amount will be the amount we deposit in our futures account, allowed coins would be the number of coin pairs we can trade with at a time, since it will be dependent on our trade amount, we need to set a limit, you could remove trade amount, and get it from the exchanges' API, but I have added to the DB since it would save us a network call to the exchange. Capital per trade is the amount for opening one position on the exchange. So, for example, I can set my trade amount to be $600, and I set the allowed coins to be 3, and capital per trade to be $20; that would mean, at a single time 3 coin pair positions will be opened on the exchange each worth $20. Finally, I have a start flag, since I would want to temporarily stop my bot, since by default it should keep trading. The rest of the fields are self explanatory :wink:.

Next, we'll initialize our database and server. We'll be needing some API endpoints as well since we need to do some trivial stuff, like adding keys for now.

```Go
// controllers/server.go
package controllers

import (
	"fmt"
	"log"
	"net/http"

	"github.com/gorilla/mux"
	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/postgres" //postgres database driver
)

type Server struct {
	DB     *gorm.DB
	Router *mux.Router
}

func (server *Server) Initialize(Dbdriver, DbUser, DbPassword, DbPort, DbHost, DbName string) {
	var err error
	DBURL := fmt.Sprintf("host=%s port=%s user=%s dbname=%s sslmode=disable password=%s", DbHost, DbPort, DbUser, DbName, DbPassword)
	server.DB, err = gorm.Open(Dbdriver, DBURL)
	if err != nil {
		log.Printf("Cannot connect to the %s database", Dbdriver)
		log.Fatal("This is the error:", err)
	} else {
		log.Printf("Connected to the %s database", Dbdriver)
	}

	server.Router = mux.NewRouter()
	server.initializeRoutes()

	server.DB.AutoMigrate(&models.Key{})
}

func (server *Server) Run(addr string) {
	log.Println("Listening on port 8080")
	log.Fatal(http.ListenAndServe("127.0.0.1"+addr, server.Router))
}
```

We haven't created the function for initializing routes, we'll do that now:

```Go
// controllers/routes.go
package controllers

func (r *Server) initializeRoutes() {
	s := r.Router.PathPrefix("/bot").Subrouter()
	s.HandleFunc("/", middleware.MiddlewareJSON(r.Home)).Methods("GET")
	s.HandleFunc("/create-key", middleware.MiddlewareJSON(r.CreateKey)).Methods("POST")
}
```

Nice, for the moment I have added a home route for testing - also for health checks - and a create key route to add an api key to the database. We'll also need some middleware functions to set headers for JSON, since it would be repeated work if we did that for each route.

```Go
// middleware/middleware.go
package middleware

import "net/http"

func MiddlewareJSON(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		r.Header.Set("Content-Type", "application/json")
		next(w, r)
	}
}
```

Before creating the route functions, let's create some functions for model, and some helper functions for encrypting and decrypting sensitive data like api key, secret key, and passphrase.

```Go
// model/key.go

func (k *Key) FindAllKeys(db *gorm.DB) (*[]Key, error) {
	Keys := []Key{}
	err := db.Model(&Key{}).Find(&Keys).Error
	if err != nil {
		return &[]Key{}, err
	}
	return &Keys, nil
}

func (k *Key) SaveKey(db *gorm.DB) (*Key, error) {
	err := db.Create(&k).Error
	if err != nil {
		return &Key{}, err
	}
	return k, nil
}

func (k *Key) FindKeysByEmail(db *gorm.DB, email string) (*[]Key, error) {
	Keys := []Key{}
	err := db.Model(Key{}).Where("user_email = ?", email).Find(&Keys).Error
	if err != nil {
		return &[]Key{}, err
	}
	return &Keys, nil
}
```

And the encryption, decryption functions:

```Go
// codec/codec.go
package codec

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"encoding/hex"
	"errors"
	"fmt"
	"io"
)

func Encrypt(key []byte, text string) (string, error) {
	block, err := aes.NewCipher(key)
	if err != nil {
		return "", err
	}
	ciphertext := make([]byte, aes.BlockSize+len(text))
	iv := ciphertext[:aes.BlockSize]
	if _, err := io.ReadFull(rand.Reader, iv); err != nil {
		return "", err
	}
	stream := cipher.NewCFBEncrypter(block, iv)
	stream.XORKeyStream(ciphertext[aes.BlockSize:], []byte(text))
	return fmt.Sprintf("%x", ciphertext), nil
}

func Decrypt(key []byte, text string) (string, error) {
	ciphertext, err := hex.DecodeString(text)
	if err != nil {
		return "", err
	}
	block, err := aes.NewCipher(key)
	if err != nil {
		return "", err
	}
	if len(ciphertext) < aes.BlockSize {
		return "", errors.New("ciphertext too short")
	}
	iv := ciphertext[:aes.BlockSize]
	ciphertext = ciphertext[aes.BlockSize:]
	stream := cipher.NewCFBDecrypter(block, iv)
	stream.XORKeyStream(ciphertext, ciphertext)
	return string(ciphertext), nil
}
```

Now, we'll create the routes for the health check and for creating the key.

```Go
// controllers/key.go
package controllers

import (
	"encoding/json"
	"errors"
	"net/http"
	"os"
)

type KeyRequest struct {
	ApiKey          string  `json:"api_key"`
	SecretKey       string  `json:"secret_key"`
	Passphrase      string  `json:"passphrase"`
	UserEmail       string  `json:"user_email"`
	TradeAmount     int     `json:"trade_amount"`
	AllowedCoins    int     `json:"allowed_coins"`
	CapitalPerTrade float64 `json:"capital_per_trade"`
	Start           bool    `json:"start"`
}

func (server *Server) CreateKey(w http.ResponseWriter, r *http.Request) {
	encryptionKey := os.Getenv("ENCRYPTION_PASS")
	var keyRequest KeyRequest
	err := json.NewDecoder(r.Body).Decode(&keyRequest)
	if err != nil {
		response.ERROR(w, http.StatusBadRequest, errors.New("bad params for key"))
		return
	}

	encryptedApiKey, err := codec.Encrypt([]byte(encryptionKey), keyRequest.ApiKey)
	if err != nil {
		response.ERROR(w, http.StatusInternalServerError, errors.New("failed to encrypt api key"))
		return
	}
	encryptedSecretKey, err := codec.Encrypt([]byte(encryptionKey), keyRequest.SecretKey)
	if err != nil {
		response.ERROR(w, http.StatusInternalServerError, errors.New("failed to encrypt secret key"))
		return
	}
	encryptedPassphrase, err := codec.Encrypt([]byte(encryptionKey), keyRequest.Passphrase)
	if err != nil {
		response.ERROR(w, http.StatusInternalServerError, errors.New("failed to encrypt passphrase"))
		return
	}

	// Create Key object
	key := &models.Key{
		ApiKey:          encryptedApiKey,
		SecretKey:       encryptedSecretKey,
		Passphrase:      encryptedPassphrase,
		UserEmail:       keyRequest.UserEmail,
		TradeAmount:     keyRequest.TradeAmount,
		AllowedCoins:    keyRequest.AllowedCoins,
		CapitalPerTrade: keyRequest.CapitalPerTrade,
		Start:           keyRequest.Start,
	}

	key, err = key.SaveKey(server.DB)
	if err != nil {
		response.ERROR(w, http.StatusInternalServerError, errors.New("failed to save key"))
		return
	}

	response.JSON(w, http.StatusOK, key)
}
```

For the health check:

```Go
// controllers/home.go
package controllers

import (
	"net/http"
)

func (server *Server) Home(w http.ResponseWriter, r *http.Request) {
	response.JSON(w, http.StatusOK, "Future Trading Bot")
}
```

You might be wondering what this response package is; since I have added some reusable logic in the middleware functions for setting headers, I have created some functions to handle sending responses as well:

```Go
package response

import (
	"encoding/json"
	"fmt"
	"net/http"
)

func JSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	err := json.NewEncoder(w).Encode(data)
	if err != nil {
		fmt.Fprintf(w, "%s", err.Error())
	}
}

func ERROR(w http.ResponseWriter, statusCode int, err error) {
	if err != nil {
		JSON(w, statusCode, struct {
			Error string `json:"error"`
		}{
			Error: err.Error(),
		})
		return
	}
	JSON(w, http.StatusBadRequest, nil)
}
```

Phew, that's alot of work for a single post xD, but we're almost done. Finally we're left with the main.go file:

```Go
package main

import (
	"log"
	"os"

	"github.com/joho/godotenv"
)

var server = controllers.Server{}

func Run() {
	err := godotenv.Load()

	if err != nil {
		log.Fatal("Error getting env")
	} else {
		log.Println("Getting Values")
	}

	server.Initialize(os.Getenv("DB_DRIVER"), os.Getenv("DB_USER"), os.Getenv("DB_PASSWORD"), os.Getenv("DB_PORT"), os.Getenv("DB_HOST"), os.Getenv("DB_NAME"))
	server.Run(":8080")
}

func main() {
	Run()
}
```

We're loading the env variables and initializing the server (and the database) and running it. Now I'll be using Bitget as ny exchange and I'll be creating the bot to interact with their API since their documentation is pretty good. You can go to their dashboard and open their API section and create a new system generated key, give it read/write permissions, allow future trading and then copy the key details. Post the key to the create key route using Postman, and we're done. In the next part we'll connect with Bitget and get some basic API operations running. Adios.




