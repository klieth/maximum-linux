require "net/http"
require "uri"
require "json"

def prompt(question)
  puts question
  STDIN.gets.chomp
end

def prompt_yes_no(question)
  answer = ''
  until %w[y n].include? answer
    answer = prompt("#{question} [y/n]") { |response| response.limit = 1; response.case = :downcase }
  end
  answer == 'y'
end

def prompt_number(question, high)
  accepted = (1..high).to_a
  answer = ''
  until accepted.include? answer
    answer = prompt("#{question} [1-#{high}]") { |response| response.limit = 1 }
    answer = answer.to_i
  end
  answer
end

def prompt_list(list)
  list.each_with_index do |item, index|
    puts "#{index + 1}. #{item}"
  end
  list[prompt_number("Select one", list.length) - 1]
end

def pacman_install(package)
  `pacman -Q #{package} 1>/dev/null 2>/dev/null`
  if $? == 0
    if prompt_yes_no("#{package} is already installed. Do you want to reinstall and try to update?")
      system("sudo -H bash -c \"pacman -S #{package}\"")
    end
  elsif
    system("sudo -H bash -c \"pacman -S #{package}\"")
  end
end

def pacman_install_group(package)
  `pacman -Qg #{package} 1>/dev/null 2>/dev/null`
  if $? == 0
    if prompt_yes_no("#{package} is already installed. Do you want to reinstall and try to update?")
      system("sudo -H bash -c \"pacman -S #{package}\"")
    end
  elsif
    system("sudo -H bash -c \"pacman -S #{package}\"")
  end
end

def aur_helper(package)
  info = JSON.parse(Net::HTTP.get(URI.parse("https://aur.archlinux.org/rpc.php?type=info&arg=#{package}")))
  system("curl https://aur.archlinux.org#{info["results"]["URLPath"]} | tar xz && cd #{package} && makepkg -s && sudo -H bash -c \"pacman -U *.pkg.*\"")
  system("ls #{package} && rm -rf #{package}")
end

def aur_install(package)
  `pacman -Q #{package} 1>/dev/null 2>/dev/null`
  if $? == 0
    if prompt_yes_no("#{package} is already installed. Do you want to reinstall and try to update?")
      aur_helper(package)
    end
  elsif
    aur_helper(package)
  end
end

def yaourt_install(package)
  `pacman -Q #{package} 1>/dev/null 2>/dev/null`
  if $? == 0
    if prompt_yes_no("#{package} is already installed. Do you want to reinstall and try to update?")
      system("yaourt -S #{package}")
    end
  elsif
    system("yaourt -S #{package}")
  end
end

def step(description)
  description = "-- #{description} "
  description = description.ljust(80, '-')
  puts
  puts "\e[32m#{description}\e[0m"
end

def get_backup_path(path)
  number = 1
  backup_path = "#{path}.bak"
  loop do
    if number > 1
      backup_path = "#{backup_path}#{number}"
    end
    if File.exists?(backup_path) || File.symlink?(backup_path)
      number += 1
      next
    end
    break
  end
  backup_path
end

def link_file(original_filename, symlink_filename)
  original_path = File.expand_path(original_filename)
  symlink_path = File.expand_path(symlink_filename)
  if File.exists?(symlink_path) || File.symlink?(symlink_path)
    if File.symlink?(symlink_path)
      symlink_points_to_path = File.readlink(symlink_path)
      return if symlink_points_to_path == original_path
      # Symlinks can't just be moved like regular files. Recreate old one, and
      # remove it.
      ln_s symlink_points_to_path, get_backup_path(symlink_path), :verbose => true
      rm symlink_path
    else
      # Never move user's files without creating backups first
      mv symlink_path, get_backup_path(symlink_path), :verbose => true
    end
  end
  ln_s original_path, symlink_path, :verbose => true
end

namespace :install do
  desc "Requires pacman"
  task :pacman do
    step 'Check for pacman'
    if system("which pacman > /dev/null")
      puts "pacman found! proceeding"
    else
      puts "This script currently only runs on ArchLinux. It would be great if you could port it to work with another distro/package manager!"
    end
  end

  desc "devtools"
  task :devtools do
    step 'base-devel'
    pacman_install_group 'base-devel'
  end

  desc "yaourt"
  task :yaourt do
    step 'yaourt'
    aur_install 'package-query'
    aur_install 'yaourt'
  end

  desc "Update or install The Silver Searcher"
  task :the_silver_searcher do
    step 'the_silver_searcher'
    pacman_install "the_silver_searcher"
  end

  desc "Install a terminal"
  task :terminal do
    step 'Install a terminal'
    case term = prompt_list(['konsole','lxterminal','skip'])
    when 'skip'
      puts "No terminal selected, skipping step"
    else
      puts "Installing #{term}"
      pacman_install term
    end
  end

  desc "ctags"
  task :ctags do
    step "ctags"
    pacman_install "ctags"
  end

  desc "tmux"
  task :tmux do
    step "tmux"
    pacman_install "tmux"
  end

  desc "vundle"
  task :vundle do
    step "vundle"
    yaourt_install "vundle-git"
    system('vim -c "BundleInstall" -c "q" -c "q"')
  end
end

COPIED_FILES = {
  'vimrc.local' => "#{ENV['HOME']}/.vimrc.local",
  'vimrc.bundles.local' => "#{ENV['HOME']}/.vimrc.bundles.local",
  'tmux.conf.local' => "#{ENV['HOME']}/.tmux.conf.local"
}
LINKED_FILES = {
  'vim' => '~/.vim',
  'tmux.conf' => '~/.tmux.conf',
  'vimrc' => '~/.vimrc',
  'vimrc.bundles' => '~/.vimrc.bundles'
}

task :install do
  Rake::Task['install:pacman'].invoke
  Rake::Task['install:devtools'].invoke
  Rake::Task['install:yaourt'].invoke
  Rake::Task['install:the_silver_searcher'].invoke
  Rake::Task['install:terminal'].invoke
  Rake::Task['install:ctags'].invoke
  Rake::Task['install:tmux'].invoke

  step 'symlink'
  LINKED_FILES.each do |orig, link|
    link_file orig, link
  end
  COPIED_FILES.each do |orig, copy|
    cp orig, copy, :verbose => true unless File.exist?(copy)
  end

  Rake::Task['install:vundle'].invoke
end

task :default => :install
