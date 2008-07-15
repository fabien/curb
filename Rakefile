# $Id$
# 
require 'rake/clean'
require 'rake/testtask'
require 'rake/rdoctask'

begin
  require 'rake/gempackagetask'
rescue LoadError
  $stderr.puts("Rubygems support disabled")
end

CLEAN.include '**/*.o'
CLEAN.include "**/*.#{Config::MAKEFILE_CONFIG['DLEXT']}"
CLOBBER.include 'doc'
CLOBBER.include '**/*.log'
CLOBBER.include '**/Makefile'
CLOBBER.include '**/extconf.h'

def announce(msg='')
  $stderr.puts msg
end

desc "Default Task (Build packages)"
task :default => :package

# Determine the current version of the software
if File.read('ext/curb.h') =~ /\s*CURB_VERSION\s*['"](\d.+)['"]/
  CURRENT_VERSION = $1
else
  CURRENT_VERSION = "0.0.0"
end

if ENV['REL']
  PKG_VERSION = ENV['REL']
else
  PKG_VERSION = CURRENT_VERSION
end

task :test_ver do
  puts PKG_VERSION
end

# Make tasks -----------------------------------------------------
MAKECMD = ENV['MAKE_CMD'] || 'make'
MAKEOPTS = ENV['MAKE_OPTS'] || ''

CURB_SO = "ext/curb_core.#{Config::MAKEFILE_CONFIG['DLEXT']}"

file 'ext/Makefile' => 'ext/extconf.rb' do
  Dir.chdir('ext') do
    ruby "extconf.rb #{ENV['EXTCONF_OPTS']}"
  end
end

def make(target = '')
  Dir.chdir('ext') do 
    pid = system("#{MAKECMD} #{MAKEOPTS} #{target}")
    $?.exitstatus
  end    
end

# Let make handle dependencies between c/o/so - we'll just run it. 
file CURB_SO => 'ext/Makefile' do
  m = make
  fail "Make failed (status #{m})" unless m == 0
end

desc "Compile the shared object"
task :compile => [CURB_SO]

desc "Install to your site_ruby directory"
task :install => :alltests do
  m = make 'install' 
  fail "Make install failed (status #{m})" unless m == 0
end

# Test Tasks ---------------------------------------------------------
task :ta => :alltests
task :tu => :unittests
task :test => :unittests

if ENV['RELTEST']
  announce "Release task testing - not running regression tests on alltests"
  task :alltests => [:unittests]
else
  task :alltests => [:unittests, :bugtests]
end
                    
Rake::TestTask.new(:unittests) do |t|
  t.test_files = FileList['tests/tc_*.rb']
  t.verbose = false
end
                          
Rake::TestTask.new(:bugtests) do |t|
  t.test_files = FileList['tests/bug_*.rb']
  t.verbose = false
end
                          
#Rake::TestTask.new(:funtests) do |t|
  #  t.test_files = FileList['test/func_*.rb']
  #t.warning = true
  #t.warning = true
#end

task :unittests => :compile
task :bugtests => :compile

# RDoc Tasks ---------------------------------------------------------
desc "Create the RDOC documentation"
task :doc do
  ruby "doc.rb #{ENV['DOC_OPTS']}"
end

desc "Publish the RDoc documentation to project web site"
task :doc_upload => [ :doc ] do
  if ENV['RELTEST']
    announce "Release Task Testing, skipping doc upload"
  else    
    unless ENV['RUBYFORGE_ACCT']
      raise "Need to set RUBYFORGE_ACCT to your rubyforge.org user name (e.g. 'fred')"
    end
  
    require 'rake/contrib/sshpublisher'
    Rake::SshDirPublisher.new(
      "#{ENV['RUBYFORGE_ACCT']}@rubyforge.org",
      "/var/www/gforge-projects/curb",
      "doc"
    ).upload
  end
end

# Packaging ------------------------------------------------
PKG_FILES = FileList[
  'ext/*.rb',
  'ext/*.c',
  'ext/*.h',
  'tests/**/*',
  'samples/**/*',
  'doc.rb',
  '[A-Z]*',
]

if ! defined?(Gem)
  warn "Package Target requires RubyGEMs"
else
  spec = Gem::Specification.new do |s|
    
    #### Basic information.

    s.name = 'curb'
    s.version = PKG_VERSION
    s.summary = "Ruby bindings for the libcurl(3) URL transfer library."
    s.description = <<-EOF
      C-language Ruby bindings for the libcurl(3) URL transfer library.
    EOF
    s.extensions = 'ext/extconf.rb'    

    #### Which files are to be included in this gem? 

    s.files = PKG_FILES.to_a

    #### Load-time details
    s.require_path = 'lib'
    
    #### Documentation and testing.
    s.has_rdoc = true
    s.extra_rdoc_files = Dir['ext/*.c'] << 'ext/curb.rb' << 'README' << 'LICENSE'
    s.rdoc_options <<
      '--title' <<  'Curb API' <<
      '--main' << 'README'

    s.test_files = Dir.glob('tests/tc_*.rb')
    
    #### Author and project details.

    s.author = "Ross Bamford"
    s.email = "curb-devel@rubyforge.org"
    s.homepage = "http://curb.rubyforge.org"
    s.rubyforge_project = "curb"
  end
  
  # Quick fix for Ruby 1.8.3 / YAML bug
  if (RUBY_VERSION == '1.8.3')
    def spec.to_yaml
      out = super
      out = '--- ' + out unless out =~ /^---/
      out
    end  
  end

  package_task = Rake::GemPackageTask.new(spec) do |pkg|
    pkg.need_zip = true
    pkg.need_tar_gz = true
    pkg.package_dir = 'pkg'    
  end      
end


# --------------------------------------------------------------------
# Creating a release
desc "Make a new release (Requires SVN commit / webspace access)"
task :release => [
  :prerelease,
  :clobber,
  :alltests,
  :update_version,
  :package,
  :tag,
  :doc_upload] do
  
  announce 
  announce "**************************************************************"
  announce "* Release #{PKG_VERSION} Complete."
  announce "* Packages ready to upload."
  announce "**************************************************************"
  announce 
end

# Validate that everything is ready to go for a release.
task :prerelease do
  announce 
  announce "**************************************************************"
  announce "* Making RubyGem Release #{PKG_VERSION}"
  announce "* (current version #{CURRENT_VERSION})"
  announce "**************************************************************"
  announce  

  # Is a release number supplied?
  unless ENV['REL']
    fail "Usage: rake release REL=x.y.z [REUSE=tag_suffix]"
  end

  # Is the release different than the current release.
  # (or is REUSE set?)
  if PKG_VERSION == CURRENT_VERSION && ! ENV['REUSE']
    fail "Current version is #{PKG_VERSION}, must specify REUSE=tag_suffix to reuse version"
  end

  # Are all source files checked in?
  if ENV['RELTEST']
    announce "Release Task Testing, skipping checked-in file test"
  else
    announce "Checking for unchecked-in files..."
    data = `svn status`
    unless data =~ /^$/
      fail "SVN status is not clean ... do you have unchecked-in files?"
    end
    announce "No outstanding checkins found ... OK"
  end
  
  announce "Doc will try to use GNU cpp if available"
  ENV['DOC_OPTS'] = "--cpp"
end

# Used during release packaging if a REL is supplied
task :update_version do
  unless PKG_VERSION == CURRENT_VERSION
    pkg_vernum = PKG_VERSION.tr('.','').sub(/^0*/,'')
    pkg_vernum << '0' until pkg_vernum.length > 2
    
    File.open('ext/curb.h.new','w+') do |f|      
    maj, min, mic, patch = /(\d+)\.(\d+)(?:\.(\d+))?(?:\.(\d+))?/.match(PKG_VERSION).captures
      f << File.read('ext/curb.h').
           gsub(/CURB_VERSION\s+"(\d.+)"/) { "CURB_VERSION   \"#{PKG_VERSION}\"" }.
           gsub(/CURB_VER_NUM\s+\d+/) { "CURB_VER_NUM   #{pkg_vernum}" }.
           gsub(/CURB_VER_MAJ\s+\d+/) { "CURB_VER_MAJ   #{maj}" }.
           gsub(/CURB_VER_MIN\s+\d+/) { "CURB_VER_MIN   #{min}" }.
           gsub(/CURB_VER_MIC\s+\d+/) { "CURB_VER_MIC   #{mic || 0}" }.
           gsub(/CURB_VER_PATCH\s+\d+/) { "CURB_VER_PATCH #{patch || 0}" }           
    end
    mv('ext/curb.h.new', 'ext/curb.h')     
    if ENV['RELTEST']
      announce "Release Task Testing, skipping commiting of new version"
    else
      sh %{svn commit -m "Updated to version #{PKG_VERSION}" ext/curb.h}
    end
  end
end

# "Create a new SVN tag with the latest release number (REL=x.y.z)"
task :tag => [:prerelease] do
  reltag = "curb-#{PKG_VERSION}"
  reltag << ENV['REUSE'] if ENV['REUSE']
  announce "Tagging SVN with [#{reltag}]"
  if ENV['RELTEST']
    announce "Release Task Testing, skipping SVN tagging"
  else
    # need to get current base URL
    s = `svn info`
    if s =~ /URL:\s*([^\n]*)\n/
      svnroot = $1
      if svnroot =~ /^(.*)\/trunk/i
        svnbase = $1
        sh %{svn cp #{svnroot} #{svnbase}/TAGS/#{reltag} -m "Release #{PKG_VERSION}"}
      else
        fail "Please merge to trunk before making a release"
      end
    else 
      fail "Unable to determine repository URL from 'svn info' - is this a working copy?"
    end  
  end
end