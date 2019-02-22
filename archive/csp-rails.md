# Adding CSP to Rails

Content Security Policy can be an effective way to prevent XSS attacks. If you aren’t familiar, here’s a [great intro](https://www.html5rocks.com/en/tutorials/security/content-security-policy/).

To get started with Rails, first add the header to all requests in your `ApplicationController`. We want to start by blocking content in development so we notice it, but only report it in production so nothing breaks.

```ruby
before_action :set_csp

# use constants and freeze for performance
CSP_HEADER_NAME = (Rails.env.production? ? "Content-Security-Policy-Report-Only" : "Content-Security-Policy").freeze
CSP_HEADER_VALUE = "default-src *; report-uri /csp_reports?report_only=#{CSP_HEADER_NAME.include?("Report-Only")}".freeze

def set_csp
  response.headers[CSP_HEADER_NAME] = CSP_HEADER_VALUE
end
```

## Reports

Create a model to track reports.

```sh
rails g model CspReport
```

And in the migration, do:

```ruby
class CreateCspReports < ActiveRecord::Migration
  def change
    create_table :csp_reports do |t|
      t.text :document_uri
      t.text :referrer
      t.text :violated_directive
      t.text :effective_directive
      t.text :original_policy
      t.text :blocked_uri
      t.integer :status_code
      t.text :user_agent
      t.boolean :report_only
      t.timestamp :created_at
    end
  end
end
```

Add a controller to create the reports.

```ruby
class CspReportsController < ApplicationController
  skip_before_action :verify_authenticity_token

  def create
    report = JSON.parse(request.body.read)["csp-report"]
    CspReport.create!(
      document_uri: report["document-uri"],
      referrer: report["referrer"],
      violated_directive: report["violated-directive"],
      effective_directive: report["effective-directive"],
      original_policy: report["original-policy"],
      blocked_uri: report["blocked-uri"],
      status_code: report["status-code"],
      user_agent: request.user_agent,
      report_only: params[:report_only] == "true"
    )
    head :ok
  end
end
```

Don’t forget the route.

```ruby
resources :csp_reports, only: [:create]
```

## Enforcing the Policy

Once the reports stop, you’ll want to enforce the policy in production.

```ruby
CSP_HEADER_NAME = "Content-Security-Policy".freeze
```

## Testing New Policies

You can have both an enforced policy and a report only policy, so use this to your advantage when changing policies. Make the new policy report only for a bit before enforcing it.

```ruby
before_action :set_csp_report_only

# use constants and freeze for performance
CSP_REPORT_ONLY_HEADER_NAME = "Content-Security-Policy-Report-Only".freeze
CSP_REPORT_ONLY_HEADER_VALUE = "default-src https:; report-uri /csp_reports?report_only=true".freeze

def set_csp_report_only
  response.headers[CSP_REPORT_ONLY_HEADER_NAME] = CSP_REPORT_ONLY_HEADER_VALUE
end
```
