source "https://rubygems.org"

gem "jekyll-theme-chirpy", "~> 7.0"

group :test do
  gem "html-proofer", "~> 5.0"
end

# Windows and JRuby does not include zoneinfo files
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.1", :platforms => [:mingw, :x64_mingw, :mswin]
