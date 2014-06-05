storm
=====

Translation of Apache Storm website

## Working with Site Locally

First, install Jekyll:

```
$ cd source
$ bundle install
```

To view the site locally:

```
$ bundle exec jekyll serve -w
Configuration file: /Users/thoward/github/techdocscn/storm/source/_config.yml
            Source: /Users/thoward/github/techdocscn/storm/source
       Destination: /Users/thoward/github/techdocscn/storm/source/_site
      Generating... done.
    Server address: http://0.0.0.0:4000
  Server running... press ctrl-c to stop.
```

Now the site should be available at [http://0.0.0.0:4000](http://0.0.0.0:4000).

To build the site for publishing to GitHub pages:

```
$ git checkout gh-pages
$ cd source
$ bundle exec jekyll build
Configuration file: /Users/thoward/github/techdocscn/storm/source/_config.yml
            Source: /Users/thoward/github/techdocscn/storm/source
       Destination: /Users/thoward/github/techdocscn/storm/source/_site
      Generating... done.
$ cp -r _site/* ../
$ git add .
$ git commit -m "publish site"
$ git push origin gh-pages
```
