work_dir="/var/tmp/pigebox"
@image_dir="#{work_dir}/image"
@cache_dir=work_dir

def sudo(*arguments)
  sh "sudo -H #{arguments.join(' ')}"
end

def file(name)
  File.join("files", name)
end

def image_file(name)
  File.join(@image_dir, name)
end

def install(target, *sources)
  file_sources = sources.collect { |s| s.start_with?('/') ? s : file(s) }
  sudo "cp --preserve=mode,timestamps #{file_sources.join(' ')} #{image_file(target)}"
end

def image_mkdir(directory)
  directory = image_file(directory)
  sudo "mkdir -p #{directory}" unless File.exists?(directory)
end

def image_link(source, target)
  sudo "chroot #{@image_dir} sh -c \"ln -fs #{source} #{target}\""
end

def image_prepare_run
  sudo "mount proc #{image_file('proc')} -t proc"
end

def image_unprepare_run
	sudo "umount #{image_file('proc')}"
end

class Chroot
  def initialize(root)
    @root = root
  end
  def apt_install(*packages)
    sudo "apt-get install --yes --force-yes #{packages.join(' ')}"
  end
  def sudo(command)
    super "chroot #{@root} sh -c \"#{command}\""
  end
end

def image_chroot(&block)
  begin
    image_prepare_run
    yield Chroot.new(@image_dir)
  ensure
    image_unprepare_run
  end
end

namespace :pigebox do

  task :setup do
    required_packages = %w{debootstrap mkisofs}
    sudo "apt-get install #{required_packages.join(' ')}"
  end

  task :clean do
    sudo "rm -rf #{@image_dir}"
  end

  task :bootstrap do
    additional_packages = %w{rsyslog netbase ifupdown net-tools dhcp3-client ssh alsa-utils ruby rubygems}
    sudo "debootstrap --variant=minbase --arch=i386 --include=#{additional_packages.join(',')} lenny #{@image_dir} http://localhost:9999/debian"
  end

  task :backup do
    sudo "tar -cf #{@cache_dir}/image.tar -C #{@image_dir} ."
  end

  task :restore => :clean do
    mkdir_p @image_dir
    sudo "tar -xf #{@cache_dir}/image.tar -C #{@image_dir} ."
  end

  namespace :configure do

    task :kernel_img do
      install "etc", "kernel-img.conf"
    end

    task :network do
      install "etc", "hosts", "hostname"

      install "etc/network", "network/interfaces"

      image_mkdir "root/.ssh"
      install "root/.ssh/authorized_keys2", ENV['HOME'] + "/.ssh/id_rsa.pub"
    end

    task :resolv_conf do
      image_mkdir "/var/etc"
      install "/var/etc", "/etc/resolv.conf"
      image_link "/var/etc/resolv.conf", "/etc/resolv.conf"
    end

    task :fstab do
      image_mkdir "/srv/pige"
      install "etc", "fstab"
      image_link "/proc/mounts", "/etc/mtab"
    end

    task :packages do
      packages = %w{linux-image-2.6-686 grub}
      image_chroot do |chroot|
        chroot.sudo "apt-get update"
        chroot.apt_install packages
        chroot.sudo "apt-get clean"
      end
    end

    task :grub do
      image_mkdir "boot/grub"
      install "boot/grub", "grub/menu.lst", image_file('/usr/lib/grub/i386-pc/stage2_eltorito')
    end

    task :alsa_backup do
      image_mkdir "etc/pige"
      install "etc/pige", "pige/alsa.backup.config"
      install "etc/init.d/alsa.backup", "pige/alsa.backup.init.d"
      install "etc/default/alsa.backup", "pige/alsa.backup.default"

      # TODO use a debian package
      image_chroot do |chroot|
        chroot.apt_install %w{ruby-dev build-essential rake libasound2 libsndfile1 libdaemons-ruby1.8}
        chroot.sudo "gem install --no-rdoc --no-ri ffi bones newgem cucumber SyslogLogger"
        chroot.sudo "gem install --no-rdoc --no-ri --source=http://gems.github.com albanpeignier-alsa-backup"

        chroot.sudo "ln -fs /var/lib/gems/1.8/bin/alsa.backup /usr/bin/alsa.backup"
        chroot.sudo "ln -fs /usr/lib/libasound.so.2.0.0 /usr/lib/libasound.so"
        chroot.sudo "ln -fs /usr/lib/libsndfile.so.1.0.17 /usr/lib/libsndfile.so"
        chroot.sudo "update-rc.d alsa.backup defaults"
      end

    end

  end

  task :configure => %w{kernel_img resolv_conf network fstab packages grub alsa_backup}.map { |t| "configure:"+t }

  namespace :dist do

    task :iso do
      sudo "mkisofs -quiet -R -b boot/grub/stage2_eltorito -no-emul-boot -boot-load-size 4 -boot-info-table -o #{@cache_dir}/pigebox.iso #{@image_dir}"
    end

  end

  task :rebuild => [ :clean, :bootstrap, :configure, "dist:iso" ]

end