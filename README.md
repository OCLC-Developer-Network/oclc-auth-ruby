# OCLC::Auth

This gem is a ruby wrapper around the Web Service Authentication system used by OCLC web services. 

## Installation

```bash
$ git clone https://github.com/OCLC-Developer-Network/oclc-auth-ruby.git
$ cd oclc-auth-ruby
$ gem build oclc-auth.gemspec
$ gem install oclc-auth-<VERSION-NUMBER>.gem
```

## Usage

### Example: Read bib from WorldCat Metadata API

This example reads a bibliographic record from the WorldCat Metadata API using the WSKey class to generate 
an HMAC signature for the authorization header.

```ruby
#!/usr/bin/env ruby

require 'net/http'
require 'oclc/auth'

wskey = OCLC::Auth::WSKey.new('api-key', 'api-key-secret')

url = 'https://worldcat.org/bib/data/823520553?classificationScheme=LibraryOfCongress&holdingLibraryCode=MAIN'
uri = URI.parse(url)

request = Net::HTTP::Get.new(uri.request_uri)
request['Authorization'] = wskey.hmac_signature('GET', url, :principal_id => 'principal-ID', :principal_idns => 'principal-IDNS')

http = Net::HTTP.new(uri.host, uri.port)
http.use_ssl = true
response = http.start do |http| 
  http.request(request)
end

puts response.body
```

### Example: Read bib from WorldCat Metadata API - Access Token via Client Credential Grant

This example reads a bibliographic record from the WorldCat Metadata API using the WSKey class to generate 
an HMAC signature for the authorization header.

```ruby
#!/usr/bin/env ruby

require 'net/http'
require 'oclc/auth'

services = ['WMS_Availability']
wskey = OCLC::Auth::WSKey.new(key, secret, :redirect_uri => redirect_uri, :services => services)

url = 'https://worldcat.org/bib/data/823520553?classificationScheme=LibraryOfCongress&holdingLibraryCode=MAIN'
uri = URI.parse(url)

accessToken = wskey.client_credentials_token(128807, 128807, :principal_id => 'principal-ID', :principal_idns => 'principal-IDNS')

request = Net::HTTP::Get.new(uri.request_uri)
request['Authorization'] = 'Bearer ' + accessToken

http = Net::HTTP.new(uri.host, uri.port)
http.use_ssl = true
response = http.start do |http| 
  http.request(request)
end

puts response.body
```

### Example: Sinatra classic-style app protected by an OAuth 2 Explicit Authorization login

In this simple [Sinatra](http://www.sinatrarb.com/) web application application, the /admin path 
is protected by an OAuth 2 Explicit Authoirzation login. Run the app with:

```bash
$ ruby webapp.rb
```

And then point your web browser at the unauthenticated URL:

[http://localhost:4567](http://localhost:4567)

and the authenticated URL:

[http://localhost:4567/admin](http://localhost:4567/admin)

to see the application behavior.

```ruby
# webapp.rb

require 'sinatra'
require 'oclc/auth'
enable :sessions

# Load the Web Service Key into the Sinatra settings object.
# This scripts assumes a key that has the specified redirect URI and 
# access to the WMS Availability web service
configure do
  key = 'api-key'
  secret = 'api-key-secret'
  redirect_uri = 'http://localhost:4567/catch_auth_code'
  services = ['WMS_Availability']
  wskey = OCLC::Auth::WSKey.new(key, secret, :redirect_uri => redirect_uri, :services => services)
  set(:wskey, wskey)
end

# Before filter to check the user's session for a valid token before rendering the admin page
before '/admin' do
  authenticate
end

get '/' do
  "Not logged in."
end

# This page catches the OAuth login redirect back to the application. This should
# match the the redirect URI on the WSKey being used.
get '/catch_auth_code' do
  if params and params[:code]
    wskey = settings.wskey
    session[:token] = wskey.auth_code_token(params[:code], 128807, 128807)
    redirect '/admin'
  else
    "This view will only render if there is an error in the login flow. " + 
    "This page renders after the browser is redirected  back to the this app with an " + 
    "error message as a URL parameter."
  end
end

# This is the resource being protected by the OAuth 2 login.
get '/admin' do
  "Logged in with access token: #{session[:token].value}"
end

# This method is called prior to rendering the admin page. It looks for the 
# presence of an OCLC::Auth::Token object in the user's session. If it does not exist
# or if it is expired, it kicks of the OAuth 2 login flow by redirecting the browser 
# to the the OCLC OAuth Authorization Server.
def authenticate
  if session[:token].nil? or session[:token].expired?
    wskey = settings.wskey
    login_url = wskey.login_url(128807, 128807)
    redirect login_url
  end
end
```
