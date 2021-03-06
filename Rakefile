require 'rubygems'
require 'rake'
require 'rake/testtask'
require 'rake/rdoctask'
require 'rake/gempackagetask'
require 'rake/contrib/rubyforgepublisher'
require File.dirname(__FILE__) + '/lib/shipping'

PKG_VERSION = Shipping::VERSION
PKG_NAME = "shipping"
PKG_FILE_NAME = "#{PKG_NAME}-#{PKG_VERSION}"
RUBY_FORGE_PROJECT = "shipping"
RUBY_FORGE_USER = ENV['RUBY_FORGE_USER'] || "cardmagic"
RELEASE_NAME = "#{PKG_NAME}-#{PKG_VERSION}"

PKG_FILES = FileList[
    "lib/**/*", "bin/*", "test/**/*", "[A-Z]*", "Rakefile", "doc/**/*"
]

desc "Default Task"
task :default => [ :test ]

# Run the unit tests
desc "Run all unit tests"
Rake::TestTask.new("test") { |t|
  t.libs << "lib"
  t.pattern = 'test/*/*_test.rb'
  t.verbose = true
}

# Make a console, useful when working on tests
desc "Generate a test console"
task :console do
   verbose( false ) { sh "irb -I lib/ -r 'shipping'" }
end

# Genereate the RDoc documentation
desc "Create documentation"
Rake::RDocTask.new("doc") { |rdoc|
  rdoc.title = "Ruby Shipping - UPS, FedEx, USPS"
  rdoc.rdoc_dir = 'html'
  rdoc.rdoc_files.include('README')
  rdoc.rdoc_files.include('lib/**/*.rb')
}

# Genereate the package
spec = Gem::Specification.new do |s|

  #### Basic information.

  s.name = 'shipping'
  s.version = PKG_VERSION
  s.summary = <<-EOF
   A general shipping module to find out the shipping prices via UPS or FedEx.
  EOF
  s.description = <<-EOF
   A general shipping module to find out the shipping prices via UPS or FedEx.
  EOF

  #### Which files are to be included in this gem?  Everything!  (Except CVS directories.)

  s.files = PKG_FILES

  #### Load-time details: library and application (you will need one or both).

  s.add_dependency('builder', '>= 1.2.0')
  s.requirements << "An xml-builder library."

  s.require_path = 'lib'
  s.autorequire = 'shipping'

  #### Documentation and testing.

  s.has_rdoc = true

  #### Author and project details.

  s.author = "Lucas Carlson"
  s.email = "lucas@rufy.com"
  s.homepage = "http://shipping.rufy.com/"
end

Rake::GemPackageTask.new(spec) do |pkg|
  pkg.need_zip = true
  pkg.need_tar = true
end

desc "Report code statistics (KLOCs, etc) from the application"
task :stats do
  require 'code_statistics'
  CodeStatistics.new(
    ["Library", "lib"],
    ["Units", "test"]
  ).to_s
end

desc "Publish new documentation"
task :publish do
    Rake::RubyForgePublisher.new('shipping', 'cardmagic').upload
    `ssh rufy update-shipping-doc`
end

desc "Publish the release files to RubyForge."
task :upload => [:package] do
    files = ["gem", "tgz", "zip"].map { |ext| "pkg/#{PKG_FILE_NAME}.#{ext}" }

    if RUBY_FORGE_PROJECT then
        require 'net/http'
        require 'open-uri'

        project_uri = "http://rubyforge.org/projects/#{RUBY_FORGE_PROJECT}/"
        project_data = open(project_uri) { |data| data.read }
        group_id = project_data[/[?&]group_id=(\d+)/, 1]
        raise "Couldn't get group id" unless group_id

        # This echos password to shell which is a bit sucky
        if ENV["RUBY_FORGE_PASSWORD"]
            password = ENV["RUBY_FORGE_PASSWORD"]
        else
            password = Proc.new do
              sync = STDOUT.sync
              begin
                echo false
                STDOUT.sync = true
                print "#{RUBY_FORGE_USER}@rubyforge.org's password: "
                STDIN.gets.chomp
              ensure
                echo true
                STDOUT.sync = sync
                puts
              end
            end.call
        end

        login_response = Net::HTTP.start("rubyforge.org", 80) do |http|
            data = [
                "login=1",
                "form_loginname=#{RUBY_FORGE_USER}",
                "form_pw=#{password}"
            ].join("&")
            http.post("/account/login.php", data)
        end

        cookie = login_response["set-cookie"]
        raise "Login failed" unless cookie
        headers = { "Cookie" => cookie }

        release_uri = "http://rubyforge.org/frs/admin/?group_id=#{group_id}"
        release_data = open(release_uri, headers) { |data| data.read }
        package_id = release_data[/[?&]package_id=(\d+)/, 1]
        raise "Couldn't get package id" unless package_id

        first_file = true
        release_id = ""

        files.each do |filename|
            basename  = File.basename(filename)
            file_ext  = File.extname(filename)
            file_data = File.open(filename, "rb") { |file| file.read }

            puts "Releasing #{basename}..."

            release_response = Net::HTTP.start("rubyforge.org", 80) do |http|
                release_date = Time.now.strftime("%Y-%m-%d %H:%M")
                type_map = {
                    ".zip"    => "3000",
                    ".tgz"    => "3110",
                    ".gz"     => "3110",
                    ".gem"    => "1400"
                }; type_map.default = "9999"
                type = type_map[file_ext]
                boundary = "rubyqMY6QN9bp6e4kS21H4y0zxcvoor"

                query_hash = if first_file then
                  {
                    "group_id" => group_id,
                    "package_id" => package_id,
                    "release_name" => RELEASE_NAME,
                    "release_date" => release_date,
                    "type_id" => type,
                    "processor_id" => "8000", # Any
                    "release_notes" => "",
                    "release_changes" => "",
                    "preformatted" => "1",
                    "submit" => "1"
                  }
                else
                  {
                    "group_id" => group_id,
                    "release_id" => release_id,
                    "package_id" => package_id,
                    "step2" => "1",
                    "type_id" => type,
                    "processor_id" => "8000", # Any
                    "submit" => "Add This File"
                  }
                end

                query = "?" + query_hash.map do |(name, value)|
                    [name, URI.encode(value)].join("=")
                end.join("&")

                data = [
                    "--" + boundary,
                    "Content-Disposition: form-data; name=\"userfile\"; filename=\"#{basename}\"",
                    "Content-Type: application/octet-stream",
                    "Content-Transfer-Encoding: binary",
                    "", file_data, ""
                    ].join("\x0D\x0A")

                release_headers = headers.merge(
                    "Content-Type" => "multipart/form-data; boundary=#{boundary}"
                )

                target = first_file ? "/frs/admin/qrs.php" : "/frs/admin/editrelease.php"
                http.post(target + query, data, release_headers)
            end

            if first_file then
                release_id = release_response.body[/release_id=(\d+)/, 1]
                raise("Couldn't get release id") unless release_id
            end

            first_file = false
        end
    end
end

begin
  if !defined?(USE_TERMIOS) || USE_TERMIOS
    require 'termios'
  else
    raise LoadError
  end

  # Enable or disable stdin echoing to the terminal.
  def echo(enable)
    term = Termios::getattr(STDIN)

    if enable
      term.c_lflag |= (Termios::ECHO | Termios::ICANON)
    else
      term.c_lflag &= ~Termios::ECHO
    end

    Termios::setattr(STDIN, Termios::TCSANOW, term)
  end
rescue LoadError
  def echo(enable)
  end
end
