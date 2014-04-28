task :default => :deploy

$deploy_dir = "deploy@riker.randomoracle.com:alisdair.mcdiarmid.org/"

task :deploy => :build do
  sh "rsync -rtz --delete _site/ #{$deploy_dir}"
end

task :build do
  sh "compass compile"
  sh "jekyll build"
end
