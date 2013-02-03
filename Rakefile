task :default => [:build]

task :serve => ["screen.min.css"] do
  sh "jekyll --auto --serve"
end

task :build => ["screen.min.css"] do
  sh "jekyll"
end

task :css => Dir["sass/*.sass"] do
  sh "compass compile"
end

file "screen.min.css" => [:css] do
  sh "juicer merge --force --output screen.min.css css/*.css"
end
