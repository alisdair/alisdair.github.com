task :default => [:build]

task :serve => ["css/screen.css"] do
  sh "bundle exec jekyll --auto --serve"
end

task :build => ["css/screen.css"] do
  sh "bundle exec jekyll"
end

file "css/screen.css" => Dir["sass/*.sass"] do
  sh "compass compile --output-style compressed sass/screen.sass"
end
