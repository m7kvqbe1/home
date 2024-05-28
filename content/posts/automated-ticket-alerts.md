+++
title = 'Automated Ticket Alerts'
date = 2024-05-09T12:08:10+01:00
draft = false
+++

Getting hold of resale tickets can be challenging due to high demand and rapid transactions. To tackle this, I've built an automated solution that scrapes ticket websites and sends you SMS alerts when resale tickets become available.

This was a learning exercise in concurrency in Go: goroutines, channels, wait groups etc.

Lets walk through the implementation details.

Find the related source code here:

https://github.com/m7kvqbe1/go-ticket-polling

## How the Solution Works

1. **Polling and Scraping**: The script uses the `colly` web scraping library to poll ticket websites and search for the availability of resale tickets at defined intervals.
2. **SMS Notification**: When tickets are found, the script sends an SMS to the specified phone numbers using the Textbelt API.
3. **Graceful Shutdown**: The script also listens for system signals to gracefully handle termination.

## Code Overview

Here's the core logic implemented step-by-step:

### Environment Setup

The program relies on environment variables to configure URLs, interval times, and phone numbers. It uses the `godotenv` package to load these variables from a `.env` file.

```go
func init() {
    if err := godotenv.Load(); err != nil {
        log.Fatalf("Error loading .env file: %v", err)
    }
}
```

### Scraper Struct

The `Scraper` struct manages the HTTP client, a wait group for concurrency, and a `done` channel to handle graceful exits.

```go
type Scraper struct {
    httpClient *http.Client
    waitGroup  sync.WaitGroup
    done       chan struct{}
}
```

### Sending SMS Notifications

The `sendText` method uses the Textbelt API to send SMS notifications concurrently.

```go
func (s *Scraper) sendText(number string) {
    s.waitGroup.Add(1)
    defer s.waitGroup.Done()

    key := os.Getenv("SMS_KEY")
    message := "BUY DI TIKITZ!!!"

    reqJSON, err := json.Marshal(map[string]string{
        "phone":   number,
        "message": message,
        "key":     key,
    })
    if err != nil {
        log.Println("Error encoding request body:", err)
        return
    }

    req, err := http.NewRequest(
        "POST",
        "https://textbelt.com/text",
        strings.NewReader(string(reqJSON)),
    )
    if err != nil {
        log.Println("Error creating request:", err)
        return
    }

    req.Header.Add("Content-Type", "application/json")

    resp, err := s.httpClient.Do(req)
    if err != nil {
        log.Println("Error sending SMS:", err)
        return
    }

    defer resp.Body.Close()

    if resp.StatusCode == http.StatusOK {
        log.Printf("SMS sent: %s\n", number)
    } else {
        log.Printf("Failed to send SMS to %s\n", number)
    }
}
```

### Fetching and Scraping

The `fetch` method uses the `colly` library to perform the scraping. Each scraping task runs in its own goroutine to ensure the program remains responsive and continues processing subsequent polling intervals even if one scraping task is delayed.

```go
func (s *Scraper) fetch() {
    s.waitGroup.Add(1)
    defer s.waitGroup.Done()

    c := colly.NewCollector(
        colly.UserAgent("Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/91.0.4472.124 Safari/537.36"),
    )

    c.WithTransport(&http.Transport{
        DialContext: (&net.Dialer{}).DialContext,
        TLSClientConfig: &tls.Config{
            InsecureSkipVerify: true,
        },
    })

    c.OnRequest(func(r *colly.Request) {
        r.Headers.Set("Accept", "text/html,application/xhtml+xml,application/xml;q=0.9")
        r.Headers.Set("Accept-Language", "en-US,en;q=0.9")
        r.Headers.Set("Referer", "https://www.google.com/")
        log.Println("Visiting", r.URL.String())
    })

    foundBuynow := false

    c.OnHTML("a[id='buynow']", func(e *colly.HTMLElement) {
        foundBuynow = true
        s.success()
    })

    c.OnScraped(func(r *colly.Response) {
        if !foundBuynow {
            s.failure()
        }
    })

    c.OnError(func(r *colly.Response, err error) {
        log.Printf("Error fetching %s: %v\n", r.Request.URL, err)
    })

    c.Visit(os.Getenv("URL"))
}
```

### Success and Failure Handling

In the `success` method, SMS messages are sent concurrently using goroutines for each phone number.

This parallel execution speeds things up if we've got a lot of numbers to text. The `failure` method simply logs that no tickets were found.

```go
func (s *Scraper) success() {
    log.Println("BUY DI TIKITZ!!!")

    phoneNumbers := strings.Split(os.Getenv("PHONE_NUMBERS"), ",")
    for _, number := range phoneNumbers {
        go s.sendText(number)
    }

    close(s.done)
}

func (s *Scraper) failure() {
    log.Println("no tikz found...")
}
```

### Polling Loop

The `scrapeLoop` method manages a continuous polling loop, invoking the `fetch` method at each tick of the ticker. It uses a select statement to listen to the `done` channel and gracefully exit when needed, or to start a new scraping task concurrently with the existing polling interval.

```go
func (s *Scraper) scrapeLoop() {
    intervalMS, err := strconv.Atoi(os.Getenv("INTERVAL_MS"))
    if err != nil {
        log.Fatalf("Error parsing INTERVAL_MS: %v", err)
    }

    ticker := time.NewTicker(time.Duration(intervalMS) * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-s.done:
            return
        case <-ticker.C:
            go s.fetch()
        }
    }
}
```

### Main Function

The `main` function sets up the scraper and gracefully handles system signals, making sure all concurrent processes finish before exiting.

```go
func main() {
    log.Println("Polling for da tikz...")

    scraper := &Scraper{
        httpClient: &http.Client{
            Transport: &http.Transport{
                TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
            },
        },
        done: make(chan struct{}),
    }

    signals := make(chan os.Signal, 1)
    signal.Notify(signals, syscall.SIGINT, syscall.SIGTERM)

    go scraper.scrapeLoop()

    select {
    case <-signals:
        log.Println("Received an interrupt, stopping service...")
    case <-scraper.done:
        log.Println("Success! Terminating gracefully...")
    }

    scraper.waitGroup.Wait()
    log.Println("Shutting down gracefully")
}
```
