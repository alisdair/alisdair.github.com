task :default => [:build]

task :serve => ["css/screen.css"] do
  sh "jekyll --auto --serve"
end

task :build => ["css/screen.css"] do
  sh "jekyll"
end

file "css/screen.css" => Dir["sass/*.sass"] do
  sh "compass compile --output-style compressed sass/screen.sass"
end
