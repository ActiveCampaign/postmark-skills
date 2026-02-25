# SDK Installation

## Detect Project Language

| File | Language | SDK Package | Install Command |
|------|----------|-------------|-----------------|
| `package.json` | Node.js / TypeScript | `postmark` | `npm install postmark` |
| `requirements.txt` / `pyproject.toml` | Python | `postmarker` | `pip install postmarker` |
| `Gemfile` | Ruby | `postmark` | `gem install postmark` |
| `composer.json` | PHP | `wildbit/postmark-php` | `composer require wildbit/postmark-php` |
| `*.csproj` / `*.sln` | .NET | `Postmark` | `dotnet add package Postmark` |

## Node.js / TypeScript

```bash
npm install postmark
```

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);
```

Or with ES modules:

```typescript
import { ServerClient } from 'postmark';
const client = new ServerClient(process.env.POSTMARK_SERVER_TOKEN);
```

**Source:** [github.com/ActiveCampaign/postmark.js](https://github.com/ActiveCampaign/postmark.js)

## Python

```bash
pip install postmarker
```

```python
import os
from postmarker.core import PostmarkClient

postmark = PostmarkClient(server_token=os.environ['POSTMARK_SERVER_TOKEN'])
```

**Source:** [github.com/Stranger6667/postmarker](https://github.com/Stranger6667/postmarker)

## Ruby

```bash
gem install postmark
```

```ruby
require 'postmark'

client = Postmark::ApiClient.new(ENV['POSTMARK_SERVER_TOKEN'])
```

**Source:** [github.com/ActiveCampaign/postmark-gem](https://github.com/ActiveCampaign/postmark-gem)

## PHP

```bash
composer require wildbit/postmark-php
```

```php
use Postmark\PostmarkClient;

$client = new PostmarkClient(getenv('POSTMARK_SERVER_TOKEN'));
```

**Source:** [github.com/ActiveCampaign/postmark-php](https://github.com/ActiveCampaign/postmark-php)

## .NET

```bash
dotnet add package Postmark
```

```csharp
using PostmarkDotNet;

var client = new PostmarkClient(Environment.GetEnvironmentVariable("POSTMARK_SERVER_TOKEN"));
```

**Source:** [github.com/ActiveCampaign/postmark-dotnet](https://github.com/ActiveCampaign/postmark-dotnet)

## Environment Variable

All SDKs should read the API token from the `POSTMARK_SERVER_TOKEN` environment variable:

```bash
export POSTMARK_SERVER_TOKEN=your-server-token-here
```

For testing, use the test token:

```bash
export POSTMARK_SERVER_TOKEN=POSTMARK_API_TEST
```
