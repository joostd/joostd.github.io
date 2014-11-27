For now, I use the default [Jekyll theme] (https://github.com/jglovier/jekyll-new])

On OSX, install ruby from brew, instead of relying on a ruby version from your osx distribution.

	brew install ruby

Then, install bundler using

	gem install bundler

Use the Gemfile for installing the github-pages gem:

	bundle install

Generate sample site:
	bundle exec jekyll new site

Start Jekyll's server for development:

	bundle exec jekyll server

See also
- https://help.github.com/articles/using-jekyll-with-pages/

