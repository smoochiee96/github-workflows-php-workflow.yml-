# PHP Sample Code for the Authorize.Net SDK
[![Travis CI Status](https://travis-ci.org/AuthorizeNet/sample-code-php.svg?branch=master)](https://travis-ci.org/AuthorizeNet/sample-code-php)

This repository contains working code samples which demonstrate PHP integration with the [Authorize.Net PHP SDK](https://github.com/AuthorizeNet/sdk-php).

The samples are organized into categories and common usage examples, just like our [API Reference Guide](http://developer.authorize.net/api/reference). Our API Reference Guide is an interactive reference for the Authorize.Net API. It explains the request and response parameters for each API method and has embedded code windows to allow you to send actual requests right within the API Reference Guide.


## Using the Sample Code

The samples are all completely independent and self-contained. You can analyze them to get an understanding of how a particular method works, or you can use the snippets as a starting point for your own project.

You can also run each sample directly from the command line.

## Running the Samples From the Command Line
* Clone this repository:
```
    $ git clone https://github.com/AuthorizeNet/sample-code-php.git
```
* Run composer with the "update" option in the root directory of the repository.
```
    $ composer update
```
* Run the individual samples by name. For example:
```
    $ php PaymentTransactions/[CodeSampleName]
```
e.g.
```
    $ php PaymentTransactions/authorize-credit-card.php
```

### Installation Notes
Note: If during "composer update", you get the error "composer failed to open stream invalid argument", go to your php.ini file (present where you have installed PHP), and uncomment the following lines:
```
extension=php_openssl.dll
extension=php_curl.dll
```
On Windows systems, you also have to uncomment:
```
extension_dir = "ext"
```
Then run `composer update` again. You might have to restart your machine before the changes take effect.

### What if I'm not using Composer?
We provide a custom `SPL` autoloader. Just [download the SDK](https://github.com/AuthorizeNet/sdk-php/releases) and point to its `autoload.php` file:

```php
require 'path/to/anet_php_sdk/autoload.php';
```
name: PHP SDK CI/CD Pipeline

on:
  push:
    branches: [ master, main ]
  pull_request:
    branches: [ master, main ]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  tests:
    name: PHP ${{ matrix.php-version }} - ${{ matrix.dependency-version }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        php-version: ['7.4', '8.0', '8.1', '8.2', '8.3']
        dependency-version: ['prefer-stable']

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup PHP Environment
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php-version }}
          extensions: mbstring, xml, ctype, iconv, intl, simplexml, dom, json, curl, openssl
          coverage: none
          tools: composer:v2

      - name: Get Composer Cache Directory
        id: composer-cache
        run: echo "dir=$(composer config cache-files-dir)" >> $GITHUB_OUTPUT

      - name: Cache Composer Dependencies
        uses: actions/cache@v4
        with:
          path: ${{ steps.composer-cache.outputs.dir }}
          key: ${{ runner.os }}-composer-${{ matrix.php-version }}-${{ hashFiles('**/composer.json') }}
          restore-keys: |
            ${{ runner.os }}-composer-${{ matrix.php-version }}-

      - name: Install Dependencies
        run: |
          composer update --${{ matrix.dependency-version }} --no-interaction --prefer-dist --optimize-autoloader

      - name: Configure Test Environment
        run: |
          if [ -f phpunit.xml.dist ]; then
            cp phpunit.xml.dist phpunit.xml
          fi

      - name: Run PHPUnit Test Suite
        env:
          MERCHANT_LOGIN_ID: ${{ secrets.AUTHNET_MERCHANT_LOGIN_ID }}
          MERCHANT_TRANSACTION_KEY: ${{ secrets.AUTHNET_MERCHANT_TRANSACTION_KEY }}
        run: |
          if [ -f vendor/bin/phpunit ]; then
            vendor/bin/phpunit --colors=always
          else
            echo "No PHPUnit binary detected in vendor/bin. Run composer require --dev phpunit/phpunit."
            exit 1
          fi

  lint:
    name: Code Quality and Linting
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          tools: composer:v2

      - name: Validate composer.json
        run: composer validate --strict --no-check-publish

      - name: Install Dependencies
        run: composer install --prefer-dist --no-progress
