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