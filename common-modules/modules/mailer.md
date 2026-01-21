# mailer

SMTP email sending with TLS support using Gomail.

## Installation

```bash
go get github.com/weedbox/common-modules/mailer
```

## Usage in Fx

```go
import "github.com/weedbox/common-modules/mailer"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        mailer.Module("mailer"),
    }, nil
}
```

## Configuration

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.host` | `0.0.0.0` | SMTP host |
| `{scope}.port` | `25` | SMTP port |
| `{scope}.username` | (empty) | SMTP username |
| `{scope}.password` | (empty) | SMTP password |
| `{scope}.tls` | `false` | Enable TLS |

### TOML Example

```toml
[mailer]
host = "smtp.gmail.com"
port = 587
username = "noreply@example.com"
password = "app-password"
tls = true
```

### Environment Variables

```bash
export MAILER_HOST=smtp.gmail.com
export MAILER_PORT=587
export MAILER_USERNAME=noreply@example.com
export MAILER_PASSWORD=app-password
export MAILER_TLS=true
```

### Common SMTP Settings

| Provider | Host | Port | TLS |
|----------|------|------|-----|
| Gmail | smtp.gmail.com | 587 | true |
| Outlook | smtp.office365.com | 587 | true |
| SendGrid | smtp.sendgrid.net | 587 | true |
| Mailgun | smtp.mailgun.org | 587 | true |
| Amazon SES | email-smtp.{region}.amazonaws.com | 587 | true |

## Basic Usage

```go
import "github.com/weedbox/common-modules/mailer"

type Params struct {
    fx.In
    Mailer *mailer.Mailer
}

type MyModule struct {
    params Params
}

func (m *MyModule) SendEmail(to, subject, body string) error {
    msg := m.params.Mailer.NewMessage()

    msg.SetHeader("From", "noreply@example.com")
    msg.SetHeader("To", to)
    msg.SetHeader("Subject", subject)
    msg.SetBody("text/plain", body)

    return m.params.Mailer.Send(msg)
}
```

## Plain Text Email

```go
func (m *MyModule) SendWelcomeEmail(to, name string) error {
    msg := m.params.Mailer.NewMessage()

    msg.SetHeader("From", "noreply@example.com")
    msg.SetHeader("To", to)
    msg.SetHeader("Subject", "Welcome!")
    msg.SetBody("text/plain", "Hello "+name+",\n\nWelcome to our service!\n\nBest regards,\nThe Team")

    return m.params.Mailer.Send(msg)
}
```

## HTML Email

```go
func (m *MyModule) SendHTMLEmail(to, name string) error {
    msg := m.params.Mailer.NewMessage()

    msg.SetHeader("From", "noreply@example.com")
    msg.SetHeader("To", to)
    msg.SetHeader("Subject", "Welcome!")
    msg.SetBody("text/html", `
        <html>
        <body>
            <h1>Hello `+name+`</h1>
            <p>Welcome to our service!</p>
            <p>Best regards,<br>The Team</p>
        </body>
        </html>
    `)

    return m.params.Mailer.Send(msg)
}
```

## Multiple Recipients

```go
func (m *MyModule) SendToMultiple(recipients []string, subject, body string) error {
    msg := m.params.Mailer.NewMessage()

    msg.SetHeader("From", "noreply@example.com")
    msg.SetHeader("To", recipients...)
    msg.SetHeader("Subject", subject)
    msg.SetBody("text/plain", body)

    return m.params.Mailer.Send(msg)
}
```

## CC and BCC

```go
func (m *MyModule) SendWithCopy(to, cc, bcc, subject, body string) error {
    msg := m.params.Mailer.NewMessage()

    msg.SetHeader("From", "noreply@example.com")
    msg.SetHeader("To", to)
    msg.SetHeader("Cc", cc)
    msg.SetHeader("Bcc", bcc)
    msg.SetHeader("Subject", subject)
    msg.SetBody("text/plain", body)

    return m.params.Mailer.Send(msg)
}
```

## Attachments

```go
func (m *MyModule) SendWithAttachment(to, subject, body, filePath string) error {
    msg := m.params.Mailer.NewMessage()

    msg.SetHeader("From", "noreply@example.com")
    msg.SetHeader("To", to)
    msg.SetHeader("Subject", subject)
    msg.SetBody("text/plain", body)
    msg.Attach(filePath)

    return m.params.Mailer.Send(msg)
}

// Multiple attachments
func (m *MyModule) SendWithMultipleAttachments(to, subject, body string, files []string) error {
    msg := m.params.Mailer.NewMessage()

    msg.SetHeader("From", "noreply@example.com")
    msg.SetHeader("To", to)
    msg.SetHeader("Subject", subject)
    msg.SetBody("text/plain", body)

    for _, file := range files {
        msg.Attach(file)
    }

    return m.params.Mailer.Send(msg)
}
```

## Embedded Images

```go
func (m *MyModule) SendWithEmbeddedImage(to, subject string) error {
    msg := m.params.Mailer.NewMessage()

    msg.SetHeader("From", "noreply@example.com")
    msg.SetHeader("To", to)
    msg.SetHeader("Subject", subject)
    msg.SetBody("text/html", `
        <html>
        <body>
            <h1>Check out this image!</h1>
            <img src="cid:logo.png" alt="Logo">
        </body>
        </html>
    `)
    msg.Embed("/path/to/logo.png")

    return m.params.Mailer.Send(msg)
}
```

## Reply-To Header

```go
func (m *MyModule) SendWithReplyTo(to, replyTo, subject, body string) error {
    msg := m.params.Mailer.NewMessage()

    msg.SetHeader("From", "noreply@example.com")
    msg.SetHeader("Reply-To", replyTo)
    msg.SetHeader("To", to)
    msg.SetHeader("Subject", subject)
    msg.SetBody("text/plain", body)

    return m.params.Mailer.Send(msg)
}
```

## Custom From Name

```go
func (m *MyModule) SendWithFromName(to, subject, body string) error {
    msg := m.params.Mailer.NewMessage()

    msg.SetAddressHeader("From", "noreply@example.com", "My Application")
    msg.SetHeader("To", to)
    msg.SetHeader("Subject", subject)
    msg.SetBody("text/plain", body)

    return m.params.Mailer.Send(msg)
}
```

## Email Templates

```go
import (
    "bytes"
    "html/template"
)

type WelcomeData struct {
    Name      string
    LoginURL  string
    SupportEmail string
}

var welcomeTemplate = template.Must(template.New("welcome").Parse(`
<html>
<body>
    <h1>Welcome, {{.Name}}!</h1>
    <p>Your account has been created successfully.</p>
    <p><a href="{{.LoginURL}}">Click here to login</a></p>
    <p>Need help? Contact us at {{.SupportEmail}}</p>
</body>
</html>
`))

func (m *MyModule) SendWelcomeTemplate(to, name string) error {
    var buf bytes.Buffer
    data := WelcomeData{
        Name:         name,
        LoginURL:     "https://example.com/login",
        SupportEmail: "support@example.com",
    }

    if err := welcomeTemplate.Execute(&buf, data); err != nil {
        return err
    }

    msg := m.params.Mailer.NewMessage()
    msg.SetHeader("From", "noreply@example.com")
    msg.SetHeader("To", to)
    msg.SetHeader("Subject", "Welcome to Our Service!")
    msg.SetBody("text/html", buf.String())

    return m.params.Mailer.Send(msg)
}
```

## Error Handling

```go
func (m *MyModule) SendEmailWithRetry(to, subject, body string) error {
    var lastErr error

    for i := 0; i < 3; i++ {
        msg := m.params.Mailer.NewMessage()
        msg.SetHeader("From", "noreply@example.com")
        msg.SetHeader("To", to)
        msg.SetHeader("Subject", subject)
        msg.SetBody("text/plain", body)

        if err := m.params.Mailer.Send(msg); err != nil {
            lastErr = err
            m.logger.Warn("Failed to send email, retrying...",
                zap.Error(err),
                zap.Int("attempt", i+1),
            )
            time.Sleep(time.Duration(i+1) * time.Second)
            continue
        }

        return nil
    }

    return fmt.Errorf("failed to send email after 3 attempts: %w", lastErr)
}
```

## Gmail Setup

For Gmail, you need to:
1. Enable 2-Step Verification
2. Create an App Password (Security > App passwords)
3. Use the app password (not your regular password)

```toml
[mailer]
host = "smtp.gmail.com"
port = 587
username = "your-email@gmail.com"
password = "your-app-password"
tls = true
```
