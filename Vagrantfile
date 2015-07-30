# -*- mode: ruby -*-
# vi: set ft=ruby :


# puts intro
puts "\n"
title = "Catapult Release Management - https://github.com/devopsgroup-io/catapult-release-management"
length = title.size
padding = 5
puts "+".ljust(padding,"-") + "".ljust(length,"-") + "+".rjust(padding,"-")
puts "|".ljust(padding)     + title                + "|".rjust(padding)
puts "+".ljust(padding,"-") + "".ljust(length,"-") + "+".rjust(padding,"-")
puts "\n"


# libraries
require "fileutils"
require "json"
require "net/ssh"
require "net/http"
require "nokogiri"
require "open-uri"
require "openssl"
require "securerandom"
require "socket"
require "yaml"


# format errors
def catapult_exception(error)
  begin
    raise error
  rescue => exception
    puts "\n\n"
    title = "Catapult Error:"
    length = title.size
    padding = 5
    puts "+".ljust(padding,"!") + "".ljust(length,"!") + "+".rjust(padding,"!")
    puts "|".ljust(padding)     + title                + "|".rjust(padding)
    puts "+".ljust(padding,"!") + "".ljust(length,"!") + "+".rjust(padding,"!")
    puts "\n"
    puts exception.message
    puts "\n"
    puts "Please correct the error then re-run your vagrant command."
    puts "See https://github.com/devopsgroup-io/catapult-release-management for more information."
    puts "\n\n"
    exit 1
  end
end


# set variables based on operating system
if (RbConfig::CONFIG['host_os'] =~ /mswin|msys|mingw|cygwin|bccwin|wince|emc/)
  if File.exist?('C:\Program Files (x86)\Git\bin\git.exe')
    git = "\"C:\\Program Files (x86)\\Git\\bin\\git.exe\""
  else
    catapult_exception("Git is not installed at C:\\Program Files (x86)\\Git\\bin\\git.exe")
  end
elsif (RbConfig::CONFIG['host_os'] =~ /darwin|mac os|linux|solaris|bsd/)
  git = "git"
else
  catapult_exception("Cannot detect your operating system, please submit an issue at https://github.com/devopsgroup-io/catapult-release-management")
end


# check for vagrant plugins
unless Vagrant.has_plugin?("vagrant-digitalocean")
  catapult_exception('vagrant-digitalocean is not installed, please run "vagrant plugin install vagrant-digitalocean"')
end
unless Vagrant.has_plugin?("vagrant-hostmanager")
  catapult_exception('vagrant-hostmanager is not installed, please run "vagrant plugin install vagrant-hostmanager"')
end
unless Vagrant.has_plugin?("vagrant-vbguest")
  catapult_exception('vagrant-vbguest is not installed, please run "vagrant plugin install vagrant-vbguest"')
end


# require vm name on up and provision
if ["up","provision"].include?(ARGV[0])
  if ARGV.length == 1
    catapult_exception("You must use 'vagrant #{ARGV[0]} <name>', run 'vagrant status' to view VM <name>s.")
  end
end


# configure catapult and git
remote = `#{git} config --get remote.origin.url`
if remote.include?("devopsgroup-io/release-management.git") || remote.include?("devopsgroup-io/catapult-release-management.git")
  catapult_exception("In order to use Catapult Release Management, you must fork the repository so that the committed and encrypted configuration is unique to you! See https://github.com/devopsgroup-io/catapult-release-management for more information.")
else
  puts "Self updating Catapult:"
  branch = `#{git} rev-parse --abbrev-ref HEAD`.strip
  repo = `#{git} config --get remote.origin.url`
  repo_upstream = `#{git} config --get remote.upstream.url`
  repo_upstream_url = "https://github.com/devopsgroup-io/catapult-release-management.git"
  puts "\nYour repository: #{repo}"
  puts "Will sync from: #{repo_upstream}\n\n"
  if repo_upstream.empty?
    `#{git} remote add upstream #{repo_upstream_url}`
  else
    `#{git} remote rm upstream`
    `#{git} remote add upstream #{repo_upstream_url}`
  end
  repo_develop = `#{git} config --get branch.develop.remote`
  if repo_develop.empty?
    `#{git} fetch upstream`
    `#{git} checkout -b develop --track upstream/master`
    `#{git} pull upstream master`
  else
    `#{git} checkout develop`
    `#{git} pull upstream master`
  end
  `#{git} push origin develop`
  `#{git} checkout master`
  `#{git} pull upstream master`
  `#{git} push origin master`
  `#{git} checkout #{branch}`
end
# create a git pre-commit hook to ensure no configuration is committed to develop and only configuration is committed to master
FileUtils.mkdir_p(".git/hooks")
File.write('.git/hooks/pre-commit',
'#!/usr/bin/env ruby

if File.exist?(\'C:\Program Files (x86)\Git\bin\git.exe\')
  git = "\"C:\\Program Files (x86)\\Git\\bin\\git.exe\""
else
  git = "git"
end

branch = `#{git} rev-parse --abbrev-ref HEAD`
branch = branch.strip
staged = `#{git} diff --name-only --staged --word-diff=porcelain`
staged = staged.split($/)

if "#{branch}" == "develop"
  if staged.include?("configuration.yml.gpg")
    puts "Please commit configuration.yml.gpg on the master branch. You are on the develop branch, which is meant for contribution back to Catapult and should not contain your configuration files."
    exit 1
  end
  if staged.include?("provisioners/.ssh/id_rsa.gpg")
    puts "Please commit provisioners/.ssh/id_rsa.gpg on the master branch. You are on the develop branch, which is meant for contribution back to Catapult and should not contain your configuration files."
    exit 1
  end
  if staged.include?("provisioners/.ssh/id_rsa.pub.gpg")
    puts "Please commit provisioners/.ssh/id_rsa.pub.gpg on the master branch. You are on the develop branch, which is meant for contribution back to Catapult and should not contain your configuration files."
    exit 1
  end
elsif "#{branch}" == "master"
  unless staged.include?("configuration.yml.gpg") || staged.include?("provisioners/.ssh/id_rsa.gpg") || staged.include?("provisioners/.ssh/id_rsa.pub.gpg")
    puts "You are on the master branch, which is only meant for your configuration files (configuration.yml.gpg, provisioners/.ssh/id_rsa.gpg, provisioners/.ssh/id_rsa.pub.gpg). To contribute to Catapult, please switch to the develop branch."
    exit 1
  end
end

')
File.chmod(0777,'.git/hooks/pre-commit')


# bootstrap configuration-user.yml
# generate configuration-user.yml file if it does not exist
unless File.exist?("configuration-user.yml")
  FileUtils.cp("configuration-user.yml.template", "configuration-user.yml")
end
# parse configuration-user.yml and configuration-user.yml.template file
configuration_user = YAML.load_file("configuration-user.yml")
configuration_user_template = YAML.load_file("configuration-user.yml.template")
# ensure version is up-to-date
if configuration_user["settings"]["version"] != configuration_user_template["settings"]["version"]
  catapult_exception("Your configuration-user.yml file is out of date. To retain your settings please manually merge entries from configuration-user.yml.template to configuration-user.yml with your specific settings.\n*You may also delete your configuration-user.yml and re-run any vagrant command to have a vanilla version created.")
end
# check for required fields
if configuration_user["settings"]["gpg_key"] == nil || configuration_user["settings"]["gpg_key"].match(/\s/) || configuration_user["settings"]["gpg_key"].length < 20
  catapult_exception("Please set your team's gpg_key in configuration-user.yml - spaces are not permitted and must be at least 20 characters.")
end


puts "\n\n\nVerification of encrypted Catapult configuration files:\n\n"
if "#{branch}" == "develop"
  puts " * You are on the develop branch, this branch is automatically synced with Catapult core and is meant to contribute back to the core Catapult project."
  puts " * configuration.yml.gpg, provisioners/.ssh/id_rsa.gpg, and provisioners/.ssh/id_rsa.pub.gpg are checked out from the master branch so that you're able to develop and test."
  puts " * After you're finished on the develop branch, switch to the master branch and discard configuration.yml.gpg, provisioners/.ssh/id_rsa.gpg, and provisioners/.ssh/id_rsa.pub.gpg"
  puts "\n"
  `git checkout --force master -- configuration.yml.gpg`
  `git checkout --force master -- provisioners/.ssh/id_rsa.gpg`
  `git checkout --force master -- provisioners/.ssh/id_rsa.pub.gpg`
  `git reset -- configuration.yml.gpg`
  `git reset -- provisioners/.ssh/id_rsa.gpg`
  `git reset -- provisioners/.ssh/id_rsa.pub.gpg`
elsif "#{branch}" == "master"
  puts " * You are on the master branch, this branch is automatically synced with Catapult core and is meant to commit your unique configuration.yml.gpg, provisioners/.ssh/id_rsa.gpg, and provisioners/.ssh/id_rsa.pub.gpg configuration."
  if configuration_user["settings"]["gpg_edit"]
    puts " * GPG Edit Mode is enabled at configuration-user.yml[\"settings\"][\"gpg_edit\"], if there are changes to configuration.yml, provisioners/.ssh/id_rsa, or provisioners/.ssh/id_rsa.pub, they will be re-encrypted."
  end
  puts "\n"
  # bootstrap configuration.yml
  # initialize configuration.yml.gpg
  if File.zero?("configuration.yml.gpg")
    `gpg --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml.gpg --armor --cipher-algo AES256 --symmetric configuration.yml.template`
  end
  if configuration_user["settings"]["gpg_edit"]
    unless File.exist?("configuration.yml")
      # decrypt configuration.yml.gpg as configuration.yml
      `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml --decrypt configuration.yml.gpg`
    end
    # decrypt configuration.yml.gpg as configuration.yml.compare
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml.compare --decrypt configuration.yml.gpg`
    if FileUtils.compare_file('configuration.yml', 'configuration.yml.compare')
      puts "\n * There were no changes to configuration.yml, no need to encrypt as this would create a new cipher to commit.\n\n"
    else
      puts "\n * There were changes to configuration.yml, encrypting configuration.yml as configuration.yml.gpg. Please commit these changes to the master branch for your team to get the changes.\n\n"
      `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml.gpg --armor --cipher-algo AES256 --symmetric configuration.yml`
    end
    FileUtils.rm('configuration.yml.compare')
  else
    # decrypt configuration.yml.gpg as configuration.yml
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml --decrypt configuration.yml.gpg`
  end
  # bootstrap ssh keys
  # decrypt id_rsa and id_rsa.pub
  if File.zero?("provisioners/.ssh/id_rsa.gpg") || File.zero?("provisioners/.ssh/id_rsa.pub.gpg")
    catapult_exception("Please place your team's ssh public (id_rsa.pub) and private key (id_rsa.pub) in provisioners/.ssh")
  end
  if configuration_user["settings"]["gpg_edit"]
    unless File.exist?("provisioners/.ssh/id_rsa")
      `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output provisioners/.ssh/id_rsa --decrypt provisioners/.ssh/id_rsa.gpg`
    end
    unless File.exist?("provisioners/.ssh/id_rsa.pub")
      `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output provisioners/.ssh/id_rsa.pub --decrypt provisioners/.ssh/id_rsa.pub.gpg`
    end
    # decrypt provisioners/.ssh/id_rsa.gpg as provisioners/.ssh/id_rsa.compare
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output provisioners/.ssh/id_rsa.compare --decrypt provisioners/.ssh/id_rsa.gpg`
    if FileUtils.compare_file('provisioners/.ssh/id_rsa', 'provisioners/.ssh/id_rsa.compare')
      puts "\n * There were no changes to provisioners/.ssh/id_rsa, no need to encrypt as this would create a new cipher to commit.\n\n"
    else
      puts "\n * There were changes to provisioners/.ssh/id_rsa, encrypting provisioners/.ssh/id_rsa as provisioners/.ssh/id_rsa.gpg. Please commit these changes to the master branch for your team to get the changes.\n\n"
      `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output provisioners/.ssh/id_rsa.gpg --armor --cipher-algo AES256 --symmetric provisioners/.ssh/id_rsa`
    end
    FileUtils.rm('provisioners/.ssh/id_rsa.compare')
    # decrypt provisioners/.ssh/id_rsa.pub.gpg as provisioners/.ssh/id_rsa.pub.compare
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output provisioners/.ssh/id_rsa.pub.compare --decrypt provisioners/.ssh/id_rsa.pub.gpg`
    if FileUtils.compare_file('provisioners/.ssh/id_rsa.pub', 'provisioners/.ssh/id_rsa.pub.compare')
      puts "\n * There were no changes to provisioners/.ssh/id_rsa.pub, no need to encrypt as this would create a new cipher to commit.\n\n"
    else
      puts "\n * There were changes to provisioners/.ssh/id_rsa.pub, encrypting provisioners/.ssh/id_rsa.pub as provisioners/.ssh/id_rsa.pub.gpg. Please commit these changes to the master branch for your team to get the changes.\n\n"
      `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output provisioners/.ssh/id_rsa.pub.gpg --armor --cipher-algo AES256 --symmetric provisioners/.ssh/id_rsa.pub`
    end
    FileUtils.rm('provisioners/.ssh/id_rsa.pub.compare')
  else
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output provisioners/.ssh/id_rsa --decrypt provisioners/.ssh/id_rsa.gpg`
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output provisioners/.ssh/id_rsa.pub --decrypt provisioners/.ssh/id_rsa.pub.gpg`
  end
  
end
# create objects from configuration.yml.gpg and configuration.yml.template
configuration = YAML.load(`gpg --batch --passphrase "#{configuration_user["settings"]["gpg_key"]}" --decrypt configuration.yml.gpg`)
configuration_example = YAML.load_file("configuration.yml.template")
`gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output provisioners/.ssh/id_rsa --decrypt provisioners/.ssh/id_rsa.gpg`
`gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output provisioners/.ssh/id_rsa.pub --decrypt provisioners/.ssh/id_rsa.pub.gpg`



puts "\nVerification of configuration[\"software\"]:\n\n"
# validate configuration["software"]
if configuration["software"]["version"] != configuration_example["software"]["version"]
  catapult_exception("Your configuration.yml file is out of date. To retain your settings please manually duplicate entries from configuration.yml.template with your specific settings.\n*You may also delete your configuration.yml and re-run any vagrant command to have a vanilla version created.")
end
puts " * Status: Verified"
puts "\nVerification of configuration[\"company\"]:\n\n"
# validate configuration["company"]
if configuration["company"]["name"] == nil
  catapult_exception("Please set [\"company\"][\"name\"] in configuration.yml")
end
if configuration["company"]["email"] == nil
  catapult_exception("Please set [\"company\"][\"email\"] in configuration.yml")
end
if configuration["company"]["timezone_redhat"] == nil
  catapult_exception("Please set [\"company\"][\"timezone_redhat\"] in configuration.yml")
end
if configuration["company"]["timezone_windows"] == nil
  catapult_exception("Please set [\"company\"][\"timezone_windows\"] in configuration.yml")
end
# https://developers.digitalocean.com/documentation/v2/
if configuration["company"]["digitalocean_personal_access_token"] == nil
  catapult_exception("Please set [\"company\"][\"digitalocean_personal_access_token\"] in configuration.yml")
else
  uri = URI("https://api.digitalocean.com/v2/droplets")
  Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
    request = Net::HTTP::Get.new uri.request_uri
    request.add_field "Authorization", "Bearer #{configuration["company"]["digitalocean_personal_access_token"]}"
    response = http.request request
    if response.code.to_f.between?(399,600)
      catapult_exception("The DigitalOcean API could not authenticate, please verify [\"company\"][\"digitalocean_personal_access_token\"].")
    else
      puts " * DigitalOcean API authenticated successfully."
      @api_digitalocean = JSON.parse(response.body)
      uri = URI("https://api.digitalocean.com/v2/account/keys")
      Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
        request = Net::HTTP::Get.new uri.request_uri
        request.add_field "Authorization", "Bearer #{configuration["company"]["digitalocean_personal_access_token"]}"
        response = http.request request
        api_digitalocean_account_keys = JSON.parse(response.body)
        @api_digitalocean_account_key_name = false
        @api_digitalocean_account_key_public_key = false
        api_digitalocean_account_keys["ssh_keys"].each do |key|
          if key["name"] == "Vagrant"
            @api_digitalocean_account_key_name = true
            if "#{key["public_key"].match(/(\w*-\w*\s\S*)/)}" == "#{File.read("provisioners/.ssh/id_rsa.pub").match(/(\w*-\w*\s\S*)/)}"
              @api_digitalocean_account_key_public_key = true
            end
          end
        end
        unless @api_digitalocean_account_key_name
          catapult_exception("Could not find the SSH Key named \"Vagrant\" in DigitalOcean, please follow the Services Setup for DigitalOcean at https://github.com/devopsgroup-io/catapult-release-management#services-setup")
        else
          puts "   - Found the ssh public key \"Vagrant\""
        end
        unless @api_digitalocean_account_key_public_key
          catapult_exception("The SSH Key named \"Vagrant\" in DigitalOcean does not match your Catapult instance's SSH Key at \"provisioners/.ssh/id_rsa.pub\", please follow the Services Setup for DigitalOcean at https://github.com/devopsgroup-io/catapult-release-management#services-setup")
        else
          puts "   - The ssh public key \"Vagrant\" matches your provisioners/.ssh/id_rsa.pub ssh public key"
        end
      end
    end
  end
end
# https://confluence.atlassian.com/display/BITBUCKET/Version+1
if configuration["company"]["bitbucket_username"] == nil || configuration["company"]["bitbucket_password"] == nil
  catapult_exception("Please set [\"company\"][\"bitbucket_username\"] and [\"company\"][\"bitbucket_password\"] in configuration.yml")
else
  uri = URI("https://api.bitbucket.org/1.0/user/repositories")
  Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
    request = Net::HTTP::Get.new uri.request_uri
    request.basic_auth "#{configuration["company"]["bitbucket_username"]}", "#{configuration["company"]["bitbucket_password"]}"
    response = http.request request
    if response.code.to_f.between?(399,600)
      catapult_exception("The Bitbucket API could not authenticate, please verify [\"company\"][\"bitbucket_username\"] and [\"company\"][\"bitbucket_password\"].")
    else
      puts " * Bitbucket API authenticated successfully."
      @api_bitbucket = JSON.parse(response.body)
      # verify bitbucket user's catapult ssh key
      uri = URI("https://api.bitbucket.org/1.0/users/#{configuration["company"]["bitbucket_username"]}/ssh-keys")
      Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
        request = Net::HTTP::Get.new uri.request_uri
        request.basic_auth "#{configuration["company"]["bitbucket_username"]}", "#{configuration["company"]["bitbucket_password"]}"
        response = http.request request # Net::HTTPResponse object
        @api_bitbucket_ssh_keys = JSON.parse(response.body)
        @api_bitbucket_ssh_keys_title = false
        @api_bitbucket_ssh_keys_key = false
        unless response.code.to_f.between?(399,600)
          @api_bitbucket_ssh_keys.each do |key|
            if key["label"] == "Catapult"
              @api_bitbucket_ssh_keys_title = true
              if "#{key["key"].match(/(\w*-\w*\s\S*)/)}" == "#{File.read("provisioners/.ssh/id_rsa.pub").match(/(\w*-\w*\s\S*)/)}"
                @api_bitbucket_ssh_keys_key = true
              end
            end
          end
        end
        unless @api_bitbucket_ssh_keys_title
          catapult_exception("Could not find the SSH Key named \"Catapult\" for your Bitbucket user #{configuration["company"]["bitbucket_username"]}, please follow Provision Websites at https://github.com/devopsgroup-io/catapult-release-management#provision-websites")
        else
          puts "   - Found the ssh public key \"Catapult\" for your Bitbucket user #{configuration["company"]["bitbucket_username"]}"
        end
        unless @api_bitbucket_ssh_keys_key
          catapult_exception("The SSH Key named \"Catapult\" in Bitbucket does not match your Catapult instance's SSH Key at \"provisioners/.ssh/id_rsa.pub\", please follow Provision Websites at https://github.com/devopsgroup-io/catapult-release-management#provision-websites")
        else
          puts "   - The ssh public key \"Catapult\" matches your provisioners/.ssh/id_rsa.pub ssh public key"
        end
      end
    end
  end
end
# https://developer.github.com/v3/
if configuration["company"]["github_username"] == nil || configuration["company"]["github_password"] == nil
  catapult_exception("Please set [\"company\"][\"github_username\"] and [\"company\"][\"github_password\"] in configuration.yml")
else
  uri = URI("https://api.github.com/user")
  Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
    request = Net::HTTP::Get.new uri.request_uri
    request.basic_auth "#{configuration["company"]["github_username"]}", "#{configuration["company"]["github_password"]}"
    response = http.request request
    if response.code.to_f.between?(399,600)
      catapult_exception("The GitHub API could not authenticate, please verify [\"company\"][\"github_username\"] and [\"company\"][\"github_password\"].")
    else
      puts " * GitHub API authenticated successfully."
      @api_github = JSON.parse(response.body)
      # verify github user's catapult ssh key
      uri = URI("https://api.github.com/user/keys")
      Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
        request = Net::HTTP::Get.new uri.request_uri
        request.basic_auth "#{configuration["company"]["github_username"]}", "#{configuration["company"]["github_password"]}"
        response = http.request request # Net::HTTPResponse object
        @api_github_ssh_keys = JSON.parse(response.body)
        @api_github_ssh_keys_title = false
        @api_github_ssh_keys_key = false
        unless response.code.to_f.between?(399,600)
          @api_github_ssh_keys.each do |key|
            if key["title"] == "Catapult"
              @api_github_ssh_keys_title = true
              if "#{key["key"].match(/(\w*-\w*\s\S*)/)}" == "#{File.read("provisioners/.ssh/id_rsa.pub").match(/(\w*-\w*\s\S*)/)}"
                @api_github_ssh_keys_key = true
              end
            end
          end
        end
        unless @api_github_ssh_keys_title
          catapult_exception("Could not find the SSH Key named \"Catapult\" for your GitHub user #{configuration["company"]["github_username"]}, please follow Provision Websites at https://github.com/devopsgroup-io/catapult-release-management#provision-websites")
        else
          puts "   - Found the ssh public key \"Catapult\" for your GitHub user #{configuration["company"]["github_username"]}"
        end
        unless @api_github_ssh_keys_key
          catapult_exception("The SSH Key named \"Catapult\" in GitHub does not match your Catapult instance's SSH Key at \"provisioners/.ssh/id_rsa.pub\", please follow Provision Websites at https://github.com/devopsgroup-io/catapult-release-management#provision-websites")
        else
          puts "   - The ssh public key \"Catapult\" matches your provisioners/.ssh/id_rsa.pub ssh public key"
        end
      end
    end
  end
end
# https://docs.atlassian.com/bamboo/REST/
if configuration["company"]["bamboo_base_url"] == nil || configuration["company"]["bamboo_username"] == nil || configuration["company"]["bamboo_password"] == nil
  catapult_exception("Please set [\"company\"][\"bamboo_base_url\"] and [\"company\"][\"bamboo_username\"] and [\"company\"][\"bamboo_password\"] in configuration.yml")
else
  uri = URI("#{configuration["company"]["bamboo_base_url"]}rest/api/latest/project.json?os_authType=basic")
  Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
    request = Net::HTTP::Get.new uri.request_uri
    request.basic_auth "#{configuration["company"]["bamboo_username"]}", "#{configuration["company"]["bamboo_password"]}"
    response = http.request request
    if response.code.to_f.between?(399,600)
      catapult_exception("The Bamboo API could not authenticate, please verify [\"company\"][\"bamboo_base_url\"] and [\"company\"][\"bamboo_username\"] and [\"company\"][\"bamboo_password\"].")
    else
      puts " * Bamboo API authenticated successfully."
      @api_bamboo = JSON.parse(response.body)
      api_bamboo_project_key = @api_bamboo["projects"]["project"].find { |element| element["key"] == "CAT" }
      unless api_bamboo_project_key
        catapult_exception("Could not find the project key \"CAT\" in Bamboo, please follow the Services Setup for Bamboo at https://github.com/devopsgroup-io/catapult-release-management#services-setup")
      end
      api_bamboo_project_name = @api_bamboo["projects"]["project"].find { |element| element["name"] == "Catapult" }
      unless api_bamboo_project_name
        catapult_exception("Could not find the project name \"Catapult\" in Bamboo, please follow the Services Setup for Bamboo at https://github.com/devopsgroup-io/catapult-release-management#services-setup")
      else
        puts "   - Found the project key \"CAT\""
      end
    end
    uri = URI("#{configuration["company"]["bamboo_base_url"]}rest/api/latest/result/CAT-TEST.json?os_authType=basic")
    Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
      request = Net::HTTP::Get.new uri.request_uri
      request.basic_auth "#{configuration["company"]["bamboo_username"]}", "#{configuration["company"]["bamboo_password"]}"
      response = http.request request
      if response.code.to_f.between?(399,600)
        catapult_exception("Could not find the plan key \"TEST\" in Bamboo, please follow the Services Setup for Bamboo at https://github.com/devopsgroup-io/catapult-release-management#services-setup")
      else
        puts "   - Found the plan key \"TEST\""
      end
    end
    uri = URI("#{configuration["company"]["bamboo_base_url"]}rest/api/latest/result/CAT-QC.json?os_authType=basic")
    Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
      request = Net::HTTP::Get.new uri.request_uri
      request.basic_auth "#{configuration["company"]["bamboo_username"]}", "#{configuration["company"]["bamboo_password"]}"
      response = http.request request
      if response.code.to_f.between?(399,600)
        catapult_exception("Could not find the plan key \"QC\" in Bamboo, please follow the Services Setup for Bamboo at https://github.com/devopsgroup-io/catapult-release-management#services-setup")
      else
        puts "   - Found the plan key \"QC\""
      end
    end
    uri = URI("#{configuration["company"]["bamboo_base_url"]}rest/api/latest/result/CAT-PROD.json?os_authType=basic")
    Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
      request = Net::HTTP::Get.new uri.request_uri
      request.basic_auth "#{configuration["company"]["bamboo_username"]}", "#{configuration["company"]["bamboo_password"]}"
      response = http.request request
      if response.code.to_f.between?(399,600)
        catapult_exception("Could not find the plan key \"PROD\" in Bamboo, please follow the Services Setup for Bamboo at https://github.com/devopsgroup-io/catapult-release-management#services-setup")
      else
        puts "   - Found the plan key \"PROD\""
      end
    end
  end
end
# https://api.cloudflare.com/
if configuration["company"]["cloudflare_api_key"] == nil || configuration["company"]["cloudflare_email"] == nil
  catapult_exception("Please set [\"company\"][\"cloudflare_api_key\"] and [\"company\"][\"cloudflare_email\"] in configuration.yml")
else
  uri = URI("https://api.cloudflare.com/client/v4/zones")
  Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
    request = Net::HTTP::Get.new uri.request_uri
    request.add_field "X-Auth-Key", "#{configuration["company"]["cloudflare_api_key"]}"
    request.add_field "X-Auth-Email", "#{configuration["company"]["cloudflare_email"]}"
    response = http.request request
    if response.code.to_f.between?(399,600)
      catapult_exception("The CloudFlare API could not authenticate, please verify [\"company\"][\"cloudflare_api_key\"] and [\"company\"][\"cloudflare_email\"].")
    else
      puts " * CloudFlare API authenticated successfully."
      @api_cloudflare = JSON.parse(response.body)
    end
  end
end
puts " * Status: Verified"
puts "\nVerification of configuration[\"environments\"]:\n\n"
# validate configuration["environments"]
configuration["environments"].each do |environment,data|
  unless configuration["environments"]["#{environment}"]["servers"]["redhat_mysql"]["mysql"]["user_password"]
    configuration["environments"]["#{environment}"]["servers"]["redhat_mysql"]["mysql"]["user_password"] = SecureRandom.urlsafe_base64(16)
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml --decrypt configuration.yml.gpg`
    File.open('configuration.yml', 'w') {|f| f.write configuration.to_yaml }
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml.gpg --armor --cipher-algo AES256 --symmetric configuration.yml`
  end
  unless configuration["environments"]["#{environment}"]["servers"]["redhat_mysql"]["mysql"]["root_password"]
    configuration["environments"]["#{environment}"]["servers"]["redhat_mysql"]["mysql"]["root_password"] = SecureRandom.urlsafe_base64(16)
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml --decrypt configuration.yml.gpg`
    File.open('configuration.yml', 'w') {|f| f.write configuration.to_yaml }
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml.gpg --armor --cipher-algo AES256 --symmetric configuration.yml`
  end
  unless configuration["environments"]["#{environment}"]["software"]["drupal"]["admin_password"]
    configuration["environments"]["#{environment}"]["software"]["drupal"]["admin_password"] = SecureRandom.urlsafe_base64(16)
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml --decrypt configuration.yml.gpg`
    File.open('configuration.yml', 'w') {|f| f.write configuration.to_yaml }
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml.gpg --armor --cipher-algo AES256 --symmetric configuration.yml`
  end
  unless configuration["environments"]["#{environment}"]["software"]["wordpress"]["admin_password"]
    configuration["environments"]["#{environment}"]["software"]["wordpress"]["admin_password"] = SecureRandom.urlsafe_base64(16)
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml --decrypt configuration.yml.gpg`
    File.open('configuration.yml', 'w') {|f| f.write configuration.to_yaml }
    `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml.gpg --armor --cipher-algo AES256 --symmetric configuration.yml`
  end
  # if upstream digitalocean droplets are provisioned, get their ip addresses to write to configuration.yml
  unless environment == "dev"
    droplet = @api_digitalocean["droplets"].find { |element| element['name'] == "#{configuration["company"]["name"]}-#{environment}-redhat" }
    unless droplet == nil
      unless configuration["environments"]["#{environment}"]["servers"]["redhat"]["ip"] == droplet["networks"]["v4"].first["ip_address"]
        configuration["environments"]["#{environment}"]["servers"]["redhat"]["ip"] = droplet["networks"]["v4"].first["ip_address"]
        `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml --decrypt configuration.yml.gpg`
        File.open('configuration.yml', 'w') {|f| f.write configuration.to_yaml }
        `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml.gpg --armor --cipher-algo AES256 --symmetric configuration.yml`
      end
    end
    droplet = @api_digitalocean["droplets"].find { |element| element['name'] == "#{configuration["company"]["name"]}-#{environment}-redhat-mysql" }
    unless droplet == nil
      unless configuration["environments"]["#{environment}"]["servers"]["redhat_mysql"]["ip"] == droplet["networks"]["v4"].first["ip_address"]
        configuration["environments"]["#{environment}"]["servers"]["redhat_mysql"]["ip"] = droplet["networks"]["v4"].first["ip_address"]
        `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml --decrypt configuration.yml.gpg`
        File.open('configuration.yml', 'w') {|f| f.write configuration.to_yaml }
        `gpg --verbose --batch --yes --passphrase "#{configuration_user["settings"]["gpg_key"]}" --output configuration.yml.gpg --armor --cipher-algo AES256 --symmetric configuration.yml`
      end
    end
  end
end
puts " * Status: Verified"
puts "\nVerification of configuration[\"websites\"]:\n\n"
# validate configuration["websites"]
configuration["websites"].each do |service,data|
  domains = Array.new
  domains_sorted = Array.new
  unless configuration["websites"]["#{service}"] == nil
    puts " [#{service}]"
    configuration["websites"]["#{service}"].each do |instance|
      puts " * #{instance["domain"]}"
      # validate force_https
      unless "#{instance["force_https"]}" == ""
        unless ["true"].include?("#{instance["force_https"]}")
          catapult_exception("There is an error in your configuration.yml file.\nThe force_https for websites => #{service} => domain => #{instance["domain"]} is invalid, it must be true or removed.")
        end
      end
      # validate repo alpha order
      domains.push("#{instance["domain"]}")
      domains_sorted.push("#{instance["domain"]}")
      # validate repo format by first creating necessary split objects
      # instance["repo"] => git@github.com:devopsgroup-io/devopsgroup-io.git
      repo_split_1 = instance["repo"].split("@")
      # repo_split_1[0] => git
      # repo_split_1[1] => github.com:devopsgroup-io/devopsgroup-io.git
      repo_split_2 = repo_split_1[1].split(":")
      # repo_split_2[0] => github.com
      # repo_split_2[1] => devopsgroup-io/devopsgroup-io.git
      repo_split_3 = repo_split_2[1].split(".git")
      # repo_split_3[0] => devopsgroup-io/devopsgroup-io
      # validate repo type
      unless "#{repo_split_1[0]}" == "git"
        catapult_exception("There is an error in your configuration.yml file.\nThe repo for websites => #{service} => domain => #{instance["domain"]} is invalid, the format must be git@github.com:devopsgroup-io/devopsgroup-io.git")
      end
      # validate repo hosted at bitbucket.org or github.com
      unless "#{repo_split_2[0]}" == "bitbucket.org" || "#{repo_split_2[0]}" == "github.com"
        catapult_exception("There is an error in your configuration.yml file.\nThe repo for websites => #{service} => domain => #{instance["domain"]} is invalid, it must either be a bitbucket.org or github.com repository.")
      end
      # validate repo ends in .git
      if "#{repo_split_3[0]}" == nil
        catapult_exception("There is an error in your configuration.yml file.\nThe repo for websites => #{service} => domain => #{instance["domain"]} is invalid, it must end in .git")
      end
      # validate repo connection, this will need to be translated to windows - have fun
      if (RbConfig::CONFIG['host_os'] =~ /darwin|mac os|linux|solaris|bsd/)
        output = `ssh-agent bash -c "ssh-add provisioners/.ssh/id_rsa; git ls-remote #{instance["repo"]}" 2>/dev/null`
        output = output.split(/\n/).reject(&:empty?)
        if output.find { |element| element.include?("fatal") }
          catapult_exception("Your SSH private key for Catapult (~/provisioners/.ssh/id_rsa) cannot connect to #{instance["repo"]}, please ensure the SSH public key (~/provisioners/.ssh/id_rsa.pub) is added to this repo as a deploy key.")
        end
        unless output.find { |element| element.include?("refs/heads/master") }
          catapult_exception("A connection was established to #{instance["repo"]} using your SSH private key for Catapult (~/provisioners/.ssh/id_rsa), however, there is a no master branch, please create one.")
        end
        unless output.find { |element| element.include?("refs/heads/develop") }
          catapult_exception("A connection was established to #{instance["repo"]} using your SSH private key for Catapult (~/provisioners/.ssh/id_rsa), however, there is a no develop branch, please create one.")
        end
      end
      # create bamboo service per bitbucket repo
      if "#{repo_split_2[0]}" == "bitbucket.org"
        # the bitbucket api offers no patch for service hooks, so we first need to check if the bitbucket bamboo service hooks exist
        uri = URI("https://api.bitbucket.org/1.0/repositories/#{repo_split_3[0]}/services")
        Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
          request = Net::HTTP::Get.new uri.request_uri
          request.basic_auth "#{configuration["company"]["bitbucket_username"]}", "#{configuration["company"]["bitbucket_password"]}"
          response = http.request request # Net::HTTPResponse object
          api_bitbucket_services = JSON.parse(response.body)
          @api_bitbucket_services_bamboo_cat_test = false
          @api_bitbucket_services_bamboo_cat_qc = false
          api_bitbucket_services.each do |service|
            if service["service"]["type"] == "Bamboo"
              service["service"]["fields"].each do |field|
                if field["name"] == "Plan Key"
                  if field["value"] == "CAT-TEST"
                    @api_bitbucket_services_bamboo_cat_test = true
                  end
                  if field["value"] == "CAT-QC"
                    @api_bitbucket_services_bamboo_cat_qc = true
                  end
                end
              end
            end
          end
          unless @api_bitbucket_services_bamboo_cat_test
            uri = URI("https://api.bitbucket.org/1.0/repositories/#{repo_split_3[0]}/services")
            Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
              request = Net::HTTP::Post.new uri.request_uri
              request.basic_auth "#{configuration["company"]["bitbucket_username"]}", "#{configuration["company"]["bitbucket_password"]}"
              request.body = URI::encode\
                (""\
                  "type=Bamboo"\
                  "&URL=#{configuration["company"]["bamboo_base_url"]}"\
                  "&Plan Key=CAT-TEST"\
                  "&Username=#{configuration["company"]["bamboo_username"]}"\
                  "&Password=#{configuration["company"]["bamboo_password"]}"\
                "")
              response = http.request request # Net::HTTPResponse object
              if response.code.to_f.between?(399,600)
                catapult_exception("Unable to configure Bitbucket Bamboo service for websites => #{service} => domain => #{instance["domain"]}. Ensure the github_username defined in configuration.yml has correct access to the repository.")
              end
            end
          end
          unless @api_bitbucket_services_bamboo_cat_qc
            uri = URI("https://api.bitbucket.org/1.0/repositories/#{repo_split_3[0]}/services")
            Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
              request = Net::HTTP::Post.new uri.request_uri
              request.basic_auth "#{configuration["company"]["bitbucket_username"]}", "#{configuration["company"]["bitbucket_password"]}"
              request.body = URI::encode\
                (""\
                  "type=Bamboo"\
                  "&URL=#{configuration["company"]["bamboo_base_url"]}"\
                  "&Plan Key=CAT-QC"\
                  "&Username=#{configuration["company"]["bamboo_username"]}"\
                  "&Password=#{configuration["company"]["bamboo_password"]}"\
                "")
              response = http.request request # Net::HTTPResponse object
              if response.code.to_f.between?(399,600)
                catapult_exception("Unable to configure Bitbucket Bamboo service for websites => #{service} => domain => #{instance["domain"]}. Ensure the github_username defined in configuration.yml has correct access to the repository.")
              end
            end
          end
        end
      end
      # create bamboo service per github repo
      if "#{repo_split_2[0]}" == "github.com"
        uri = URI("https://api.github.com/repos/#{repo_split_3[0]}/hooks")
        Net::HTTP.start(uri.host, uri.port, :use_ssl => uri.scheme == 'https', :verify_mode => OpenSSL::SSL::VERIFY_NONE) do |http|
          request = Net::HTTP::Post.new uri.request_uri
          request.basic_auth "#{configuration["company"]["github_username"]}", "#{configuration["company"]["github_password"]}"
          request.body = ""\
            "{"\
              "\"name\":\"bamboo\","\
              "\"active\":true,"\
              "\"config\":"\
                "{"\
                  "\"base_url\":\"#{configuration["company"]["bamboo_base_url"]}\","\
                  "\"build_key\":\"develop:CAT-TEST,master:CAT-QC\","\
                  "\"username\":\"#{configuration["company"]["bamboo_username"]}\","\
                  "\"password\":\"#{configuration["company"]["bamboo_password"]}\""\
                "}"\
            "}"
          response = http.request request # Net::HTTPResponse object
          if response.code.to_f.between?(399,600)
            catapult_exception("Unable to configure GitHub Bamboo service for websites => #{service} => domain => #{instance["domain"]}. Ensure the github_username defined in configuration.yml has correct access to the repository.")
          end
        end
      end
      # validate software
      unless "#{instance["software"]}" == ""
        unless ["codeigniter2","drupal6","drupal7","wordpress","xenforo"].include?("#{instance["software"]}")
          catapult_exception("There is an error in your configuration.yml file.\nThe software for websites => #{service} => domain => #{instance["domain"]} is invalid, it must be one of the following [\"codeigniter2\",\"drupal6\",\"drupal7\",\"wordpress\",\"xenforo\"].")
        end
        unless ["downstream","upstream"].include?("#{instance["software_workflow"]}")
          catapult_exception("There is an error in your configuration.yml file.\nThe software for websites => #{service} => domain => #{instance["domain"]} requires the software_workflow option, it must be one of the following [\"downstream\",\"upstream\"].")
        end
      end
      # validate webroot
      unless "#{instance["webroot"]}" == ""
        unless "#{instance["webroot"]}"[-1,1] == "/"
          catapult_exception("There is an error in your configuration.yml file.\nThe webroot for websites => #{service} => domain => #{instance["domain"]} is invalid, it must include a trailing slash.")
        end
      end
    end
  end
  # ensure domains are in alpha order
  domains_sorted = domains_sorted.sort
  if domains != domains_sorted
    catapult_exception("There is an error in your configuration.yml file.\nThe domains in configuration.yml are not in alpha order for websites => #{service} - please adjust.")
  end
end
puts " * Status: Verified"


# create arrays of domains for localdev hosts file
redhathostsfile = Array.new
unless configuration["websites"]["apache"] == nil
  configuration["websites"]["apache"].each do |instance|
    redhathostsfile.push("dev.#{instance["domain"]}")
    redhathostsfile.push("www.dev.#{instance["domain"]}")
  end
end
windowshostsfile = Array.new
  unless configuration["websites"]["iis"] == nil
  configuration["websites"]["iis"].each do |instance|
    windowshostsfile.push("dev.#{instance["domain"]}")
    windowshostsfile.push("www.dev.#{instance["domain"]}")
  end
end


# vagrant status binding
if ["status"].include?(ARGV[0])
  totalwebsites = 0
  # start a new row
  puts "\n\n\nAvailable websites legend:"
  puts "\n[http response codes]"
  puts "\n * 200 ok, 301 moved permanently, 302 found, 400 bad request, 401 unauthorized, 403 forbidden, 404 not found, 500 internal server error, 502 bad gateway, 503 service unavailable, 504 gateway timeout"
  puts " * [reference] http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html"
  puts "\n[cert signature algorithm]"
  puts "\n * [reference] https://www.openssl.org/docs/apps/ciphers.html"
  puts "\n\n\nAvailable websites:"
  puts "".ljust(30) + "[software]".ljust(15) + "[dev.]".ljust(22) + "[test.]".ljust(22) + "[qc.]".ljust(22) + "[production / cert expiry, signature algorithm, common name]".ljust(80) + "[alexa rank, 3m delta]".ljust(26)

  configuration["websites"].each do |service,data|
    puts "\n[#{service}]"
    unless configuration["websites"]["#{service}"] == nil
      configuration["websites"]["#{service}"].each do |instance|
        # count websites
        totalwebsites = totalwebsites + 1
        # start new row
        row = Array.new
        # get domain name
        row.push(" * #{instance["domain"]}".ljust(29))
        # get software
        row.push((instance["software"] || "").ljust(14))
        # get http response code per environment
        configuration["environments"].each do |environment,data|
          response = nil
          if ["production"].include?("#{environment}")
            environment = nil
          else
            environment = "#{environment}."
          end
          begin
            def http_repsonse(uri_str, limit = 10)
              if limit == 0
                row.push("loop".ljust(7))
              else
                response = Net::HTTP.get_response(URI(uri_str))
                case response
                when Net::HTTPSuccess then
                  return response.code
                when Net::HTTPRedirection then
                  location = response['location']
                  http_repsonse(location, limit - 1)
                else
                  return response.code
                end
              end
            end
            row.push(http_repsonse("http://#{environment}#{instance["domain"]}").ljust(4))
          rescue SocketError
            row.push("down".ljust(4))
          rescue Errno::ECONNREFUSED
            row.push("down".ljust(4))
          rescue EOFError
            row.push("down".ljust(4))
          rescue OpenSSL::SSL::SSLError
            row.push("err".ljust(4))
          rescue Exception => ex
            row.push("#{ex.class}".ljust(4))
          end
          # nslookup production top-level domain
          begin
            row.push((Resolv.getaddress "#{environment}#{instance["domain"]}").ljust(16))
          rescue
            row.push("down".ljust(16))
          end
        end
        # ssl cert lookup
        begin 
          timeout(1) do
            tcp_client = TCPSocket.new("#{instance["domain"]}", 443)
            ssl_context = OpenSSL::SSL::SSLContext.new
            ssl_context.ssl_version = :TLSv1_2
            ssl_client = OpenSSL::SSL::SSLSocket.new(tcp_client, ssl_context)
            ssl_client.connect
            cert = OpenSSL::X509::Certificate.new(ssl_client.peer_cert)
            ssl_client.sysclose
            tcp_client.close
            #http://ruby-doc.org/stdlib-2.0/libdoc/openssl/rdoc/OpenSSL/X509/Certificate.html
            date = Date.parse((cert.not_after).to_s)
            row.push("#{date.strftime('%F')} #{cert.signature_algorithm} #{cert.subject.to_a.select{|name, _, _| name == 'CN' }.first[1]}".downcase.ljust(57))
          end
        rescue SocketError
          row.push("down".ljust(57))
        rescue Errno::ECONNREFUSED
          row.push("connection refused".ljust(57))
        rescue Errno::ECONNRESET
          row.push("connection reset".ljust(57))
        rescue Timeout::Error
          row.push("no 443 listener".ljust(57))
        rescue OpenSSL::SSL::SSLError
          row.push("cannot read cert, missing local cipher?".ljust(57))
        rescue Exception => ex
          row.push("#{ex.class}".ljust(57))
        end
        # alexa rank and 3 month deviation
        begin
          uri = URI("http://data.alexa.com/data?cli=10&url=#{instance["domain"]}")
          Net::HTTP.start(uri.host, uri.port) do |http|
            request = Net::HTTP::Get.new uri.request_uri
            response = http.request request
            response = Nokogiri::XML(response.body)
            if "#{response.xpath('//ALEXA//SD//POPULARITY')}" != ""
              response.xpath('//ALEXA//SD//POPULARITY').each do |attribute|
                row.push(attribute["TEXT"].to_s.reverse.gsub(/...(?=.)/,'\&,').reverse.ljust(11))
              end
            else
              row.push("".ljust(11))
            end
            if "#{response.xpath('//ALEXA//SD//RANK')}" != ""
              response.xpath('//ALEXA//SD//RANK').each do |attribute|
                if attribute["DELTA"].match(/\+\d+/)
                  operator = "+"
                elsif attribute["DELTA"].match(/\-\d+/)
                  operator = "-"
                end
                delta = attribute["DELTA"].split(/[+,-]/)
                row.push("#{operator}#{delta[1].to_s.reverse.gsub(/...(?=.)/,'\&,').reverse}".ljust(13))
              end
            else
              row.push("".ljust(13))
            end
          end
        end
        puts row.join(" ")
      end
    end
  end
  # start a new row
  row = Array.new
  puts "\n[total]"
  row.push(" #{totalwebsites}")
  puts row.join(" ")
end


puts "\n\n\n"


# server vms
Vagrant.configure("2") do |config|

  # vagrant hostmanager plugin configuration
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true

  # redhat localdev servers
  config.vm.define "#{configuration["company"]["name"]}-dev-redhat" do |config|
    config.vm.box = "chef/centos-7.0"
    config.vm.network "private_network", ip: configuration["environments"]["dev"]["servers"]["redhat"]["ip"]
    config.vm.network "forwarded_port", guest: 80, host: configuration["environments"]["dev"]["servers"]["redhat"]["port_80"]
    config.vm.provider :virtualbox do |provider|
      provider.memory = 512
      provider.cpus = 1
    end
    config.vm.provision :hostmanager
    config.hostmanager.aliases = redhathostsfile
    config.vm.synced_folder ".", "/vagrant", type: "nfs"
    config.vm.synced_folder "repositories", "/var/www/repositories", type: "nfs"
    config.vm.provision "shell", path: "provisioners/redhat/provision.sh", args: ["dev","#{configuration_user["settings"]["git_pull"]}","#{configuration_user["settings"]["software_validation"]}"]
  end
  config.vm.define "#{configuration["company"]["name"]}-dev-redhat-mysql" do |config|
    config.vm.box = "chef/centos-7.0"
    config.vm.network "private_network", ip: configuration["environments"]["dev"]["servers"]["redhat_mysql"]["ip"]
    config.vm.provider :virtualbox do |provider|
      provider.memory = 512
      provider.cpus = 1
    end
    config.vm.synced_folder ".", "/vagrant", type: "nfs"
    config.vm.provision :hostmanager
    config.vm.provision "shell", path: "provisioners/redhat_mysql/provision.sh", args: ["dev"]
  end

  # redhat test servers
  config.vm.define "#{configuration["company"]["name"]}-test-redhat" do |config|
    config.vm.provider :digital_ocean do |provider,override|
      override.ssh.private_key_path = "provisioners/.ssh/id_rsa"
      override.vm.box = "digital_ocean"
      override.vm.box_url = "https://github.com/smdahlen/vagrant-digitalocean/raw/master/box/digital_ocean.box"
      provider.token = configuration["company"]["digitalocean_personal_access_token"]
      provider.image = "centos-7-0-x64"
      provider.region = "nyc3"
      provider.size = "512mb"
      provider.ipv6 = true
      provider.backups_enabled = true
    end
    config.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: ["repositories/apache/*", "repositories/iis/*"]
    config.vm.provision "shell", path: "provisioners/redhat/provision.sh", args: ["test","true","false","true"]
  end
  config.vm.define "#{configuration["company"]["name"]}-test-redhat-mysql" do |config|
    config.vm.provider :digital_ocean do |provider,override|
      override.ssh.private_key_path = "provisioners/.ssh/id_rsa"
      override.vm.box = "digital_ocean"
      override.vm.box_url = "https://github.com/smdahlen/vagrant-digitalocean/raw/master/box/digital_ocean.box"
      provider.token = configuration["company"]["digitalocean_personal_access_token"]
      provider.image = "centos-7-0-x64"
      provider.region = "nyc3"
      provider.size = "512mb"
      provider.ipv6 = true
      provider.backups_enabled = true
    end
    config.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: ["repositories/apache/*", "repositories/iis/*"]
    config.vm.provision "shell", path: "provisioners/redhat_mysql/provision.sh", args: ["test"]
  end

  # redhat quality control servers
  config.vm.define "#{configuration["company"]["name"]}-qc-redhat" do |config|
    config.vm.provider :digital_ocean do |provider,override|
      override.ssh.private_key_path = "provisioners/.ssh/id_rsa"
      override.vm.box = "digital_ocean"
      override.vm.box_url = "https://github.com/smdahlen/vagrant-digitalocean/raw/master/box/digital_ocean.box"
      provider.token = configuration["company"]["digitalocean_personal_access_token"]
      provider.image = "centos-7-0-x64"
      provider.region = "nyc3"
      provider.size = "512mb"
      provider.ipv6 = true
      provider.backups_enabled = true
    end
    config.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: ["repositories/apache/*", "repositories/iis/*"]
    config.vm.provision "shell", path: "provisioners/redhat/provision.sh", args: ["qc","true","false","true"]
  end
  config.vm.define "#{configuration["company"]["name"]}-qc-redhat-mysql" do |config|
    config.vm.provider :digital_ocean do |provider,override|
      override.ssh.private_key_path = "provisioners/.ssh/id_rsa"
      override.vm.box = "digital_ocean"
      override.vm.box_url = "https://github.com/smdahlen/vagrant-digitalocean/raw/master/box/digital_ocean.box"
      provider.token = configuration["company"]["digitalocean_personal_access_token"]
      provider.image = "centos-7-0-x64"
      provider.region = "nyc3"
      provider.size = "512mb"
      provider.ipv6 = true
      provider.backups_enabled = true
    end
    config.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: ["repositories/apache/*", "repositories/iis/*"]
    config.vm.provision "shell", path: "provisioners/redhat_mysql/provision.sh", args: ["qc"]
  end

  # redhat production servers
  config.vm.define "#{configuration["company"]["name"]}-production-redhat" do |config|
    config.vm.provider :digital_ocean do |provider,override|
      override.ssh.private_key_path = "provisioners/.ssh/id_rsa"
      override.vm.box = "digital_ocean"
      override.vm.box_url = "https://github.com/smdahlen/vagrant-digitalocean/raw/master/box/digital_ocean.box"
      provider.token = configuration["company"]["digitalocean_personal_access_token"]
      provider.image = "centos-7-0-x64"
      provider.region = "nyc3"
      provider.size = "512mb"
      provider.ipv6 = true
      provider.backups_enabled = true
    end
    config.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: ["repositories/apache/*", "repositories/iis/*"]
    config.vm.provision "shell", path: "provisioners/redhat/provision.sh", args: ["production","true","false","true"]
  end
  config.vm.define "#{configuration["company"]["name"]}-production-redhat-mysql" do |config|
    config.vm.provider :digital_ocean do |provider,override|
      override.ssh.private_key_path = "provisioners/.ssh/id_rsa"
      override.vm.box = "digital_ocean"
      override.vm.box_url = "https://github.com/smdahlen/vagrant-digitalocean/raw/master/box/digital_ocean.box"
      provider.token = configuration["company"]["digitalocean_personal_access_token"]
      provider.image = "centos-7-0-x64"
      provider.region = "nyc3"
      provider.size = "512mb"
      provider.ipv6 = true
      provider.backups_enabled = true
    end
    config.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: ["repositories/apache/*", "repositories/iis/*"]
    config.vm.provision "shell", path: "provisioners/redhat_mysql/provision.sh", args: ["production"]
  end

  # windows localdev servers
  config.vm.define "#{configuration["company"]["name"]}-dev-windows" do |config|
    config.vm.box = "opentable/win-2012r2-standard-amd64-nocm"
    config.vm.network "private_network", ip: configuration["environments"]["dev"]["servers"]["windows"]["ip"]
    config.vm.network "forwarded_port", guest: 80, host: configuration["environments"]["dev"]["servers"]["redhat"]["port_80"]
    config.vm.provider :virtualbox do |provider|
      provider.memory = 512
      provider.cpus = 1
    end
    config.vm.synced_folder "repositories", "/inetpub/repositories"
    config.vm.provision :hostmanager
    config.hostmanager.aliases = windowshostsfile
    config.vm.provision "shell", path: "provisioners/windows/provision.ps1", run: "always"
    # windows specific configuration
    config.vm.guest = :windows
    config.vm.boot_timeout = 60 * 7
    config.ssh.insert_key = false
    config.vm.communicator = "winrm"
    config.vm.network "forwarded_port", guest: 3389, host: configuration["environments"]["dev"]["servers"]["redhat"]["port_3389"]
  end

end
