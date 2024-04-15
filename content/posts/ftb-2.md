+++ 
draft = false
date = 2024-04-15T12:22:51+05:00
title = "Building A Future Trading Bot: Part 2"
description = "building a future trading bot in Golang: Part 2"
slug = "ftb-2"
authors = ["Ahmed Nadeem"]
tags = ["golang", "cryptocurrency"]
categories = []
externalLink = ""
series = []
+++

In the first part we setup up our code, connected to the DB, created a route to store the API key information in the database, encrypting the keys, and finally getting the keys from the Bitget dashboard. In this part we will connect with Bitget's API, and start doing trading programmaticaly.

I'll be referencing endpoints from the [Bitget API Docs](https://www.bitget.com/api-doc/contract/intro). First we'll need to connect with Bitget. Bitget and alot of cryptocurrency exchanges use HMAC which is cryyptography technique that generates a signature by hashing the message with a secret key. It also expects a timestamp and api key in the header along with some other fields that we'll see.

Let's implement the functions for the timestamp and generating the hmac signature:

```Go
// bitget/bitget.go
func GetBitgetServerTimeStamp() string {
	return strconv.FormatInt(time.Now().UnixNano()/int64(time.Millisecond), 10)
}

func GenerateBitgetSignature(apiSecret string, method string, uri string, timestamp string, requestBody string) string {
	message := ""
	if method == "GET" {
		message = fmt.Sprintf("%s%s%s", timestamp, method, uri)
	} else if method == "POST" {
		message = fmt.Sprintf("%s%s%s%s", timestamp, method, uri, requestBody)
	}

	hmac := hmac.New(sha256.New, []byte(apiSecret))
	hmac.Write([]byte(message))
	signature := base64.StdEncoding.EncodeToString(hmac.Sum(nil))

	return signature
}
```

The process for generating a valid signature is given in the API docs, essentially, we are combining the timestamp, method, and uri into a message and signing it with our API Secret, and then converting into a base64 encoded string.

Now let's start using this function by sending a request to an API endpoint:

```Go
// bitget/bitget.go

type BitgetBatchOrderRequest struct {
	Symbol        string         `json:"symbol"`
	MarginCoin    string         `json:"marginCoin"`
	OrderDataList []OrderRequest `json:"orderDataList"`
}

type OrderRequest struct {
	Size      string `json:"size"`
	Side      string `json:"side"`
	OrderType string `json:"orderType"`
}

type BatchOrderResponse struct {
	Code        string    `json:"code"`
	Msg         string    `json:"msg"`
	RequestTime int64     `json:"requestTime"`
	Data        BatchData `json:"data"`
}

type BatchData struct {
	OrderInfo []BatchOrderInfo `json:"orderInfo"`
	Failure   []interface{}    `json:"failure"`
	Result    bool             `json:"result"`
}

type BatchOrderInfo struct {
	OrderID   string `json:"orderId"`
	ClientOid string `json:"clientOid"`
}

func PlaceBitgetBatchOrder(apiKey string, secretKey string, passphrase string, order *BitgetBatchOrderRequest) (BatchOrderResponse, error) {
	host := "https://api.bitget.com"
	path := "/api/mix/v1/order/batch-orders"
	url := host + path

	method := "POST"
	client := &http.Client{}

	jsonVal, err := json.Marshal(order)
	if err != nil {
		return BatchOrderResponse{}, err
	}

	serverTime := GetBitgetServerTimeStamp()
	signature := GenerateBitgetSignature(secretKey, "POST", path, serverTime, string(jsonVal))

	req, err := http.NewRequest(method, url, bytes.NewBuffer(jsonVal))
	req.Header.Add("ACCESS-KEY", apiKey)
	req.Header.Add("ACCESS-SIGN", signature)
	req.Header.Add("ACCESS-TIMESTAMP", serverTime)
	req.Header.Add("ACCESS-PASSPHRASE", passphrase)
	req.Header.Add("Content-Type", "application/json")
	req.Header.Add("local", "zh-CN")

	if err != nil {
		return BatchOrderResponse{}, err
	}

	res, err := client.Do(req)
	if err != nil {
		return BatchOrderResponse{}, err
	}
	defer res.Body.Close()

	body, err := ioutil.ReadAll(res.Body)
	if err != nil {
		return BatchOrderResponse{}, err
	}

	if res.StatusCode != http.StatusOK {
		return BatchOrderResponse{}, errors.New("bitget batch order failed")
	}

	var batchResponse BatchOrderResponse

	err = json.Unmarshal(body, &batchResponse)
	if err != nil {
		return BatchOrderResponse{}, err
	}

	return batchResponse, nil
}
```

We are going to create a batch order, for two positions, open and close at once - also called a hedge position. Let's create a function to test this:

```Go
func placeTrade() {
	keyModel := &models.Key{}
	key, err := keyModel.FindKeyByEmail(server.DB, "ahmedghtwhts786@gmail.com")
	if err != nil {
		log.Fatal("Error Getting key")
	}

	api_key, secret_key, passphrase := decryptKeyInfo(key.ApiKey, key.SecretKey, key.Passphrase)
	go func() {
		longOrder := bitget.OrderRequest{
			Size:      "10",
			Side:      "open_long",
			OrderType: "Market",
		}

		shortOrder := bitget.OrderRequest{
			Size:      "10",
			Side:      "open_short",
			OrderType: "Market",
		}

		batchOrderRequest := bitget.BitgetBatchOrderRequest{
			Symbol:        "SXRPSUSDT_SUMCBL",
			MarginCoin:    "SUSDT",
			OrderDataList: []bitget.OrderRequest{longOrder, shortOrder},
		}

		_, err := bitget.PlaceBitgetBatchOrder(api_key, secret_key, passphrase, &batchOrderRequest)
		if err != nil {
			log.Fatal(err)
		}
		fmt.Println("Opened Postions")
	}()

	// Asynchronously place the batch close order after a delay
	go func() {
		time.Sleep(30 * time.Second)

		longCloseOrder := bitget.OrderRequest{
			Size:      "10",
			Side:      "close_long",
			OrderType: "Market",
		}

		shortCloseOrder := bitget.OrderRequest{
			Size:      "10",
			Side:      "close_short",
			OrderType: "Market",
		}

		batchCloseOrderRequest := bitget.BitgetBatchOrderRequest{
			Symbol:        "SXRPSUSDT_SUMCBL",
			MarginCoin:    "SUSDT",
			OrderDataList: []bitget.OrderRequest{longCloseOrder, shortCloseOrder},
		}

		_, err := bitget.PlaceBitgetBatchOrder(api_key, secret_key, passphrase, &batchCloseOrderRequest)
		if err != nil {
			log.Fatal(err)
		}
		fmt.Println("Closed Postions")
	}()
}
```

I am opening and closing the positions in two goroutines, the second goroutine waits for 30 seconds before starting. This is obviously a crude way to test this but it should hopefully work. Moreover, I've done some minor refactoring, I have changed the ```findByEmail``` function to only return a single model, since for the time being we'll be using a simgle Bitget key, if there were multiple keys or exchanges involved we would need to return a slice of models. Secondly, I have created a generic function to decrypt all they key values, api key, secret key, and passphrase, and moved the config logic into the encryption function instead of passing our encryption key each.

Our code is short right now, so major refactoring is not necessary right now, but in a future blog post, we will discuss the best practices and refactor our code accordingly. 

We have successfully connected with Bitget, and started making requests to open and close positions programmatically. You could also see how we could alter this program to make money, right now I am using the demo trading version, which the docs specify as being coins with the 'S' prefix and a SUMCBL product type. Once we complete the rest of the steps, in the end we will move o real coins. In the next part we will create the conditions for trading with profit.
