# DEFINITIONS
@APP_NAME = "nway-website"
@APP_RAM  = 1024
@APP_CPU  = 1

# SETUP SCRIPT
Vagrant.configure(2) do |config|
  # ================= SETUP =================
  config.vm.box = "ubuntu/trusty64"
  config.vm.box_check_update = false
  config.vm.network "forwarded_port", guest: 3000, host: 3000
  config.vm.network "forwarded_port", guest: 5432, host: 5432
  # config.vm.network "private_network", ip: "192.168.33.10"
  # config.vm.network "public_network"
  config.vm.synced_folder "./", "/src/web-#{@APP_NAME}"

  config.vm.provider "virtualbox" do |v|
    v.memory = @APP_RAM
    v.cpus = @APP_CPU
  end

  # =========== FIX =================
  # Fix the non-tty complaint in ubuntu vagrant up/reload
  # Credits to: http://foo-o-rama.com/vagrant--stdin-is-not-a-tty--fix.html
  config.vm.provision "fix-no-tty", run: "always", type: "shell" do |s|
      s.privileged = false
      s.inline = "sudo sed -i '/tty/!s/mesg n/tty -s \\&\\& mesg n/' /root/.profile"
  end

  # =========== PROVISIONING =================
  config.vm.provision "shell", inline: <<-SHELL
    echo "Fixing VM Locale ====================================================="
    sudo locale-gen en_US.UTF-8
    export LC_ALL=en_US.UTF-8
    export LANG=en_US.UTF-8
    export LC_ADDRESS="en_US.UTF-8"
    export LC_PAPER="en_US.UTF-8"
    export LC_MONETARY="en_US.UTF-8"
    export LC_NUMERIC="en_US.UTF-8"
    export LC_TELEPHONE="en_US.UTF-8"
    export LC_IDENTIFICATION="en_US.UTF-8"
    export LC_MEASUREMENT="en_US.UTF-8"
    export LC_TIME="en_US.UTF-8"
    export LC_NAME="en_US.UTF-8"
    sudo dpkg-reconfigure locales

    echo "Update APTITUDE ======================================================"
    sudo apt-get -q update
    sudo apt-get -q -y upgrade

    echo "Setup BASIC PACKAGES ================================================="
    sudo apt-get install -q -y libpq-dev
    sudo apt-get install -q -y curl
    sudo apt-get install -q -y git

    echo "Setup RVM & RAILS ===================================================="
    gpg --quiet --batch --no-tty --yes --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
    curl -sSL https://get.rvm.io | bash -s stable --rails

    echo "Setup POSTGRESQL ====================================================="
    sudo apt-get install -q -y postgresql

    # Setup of users
    sudo -u postgres psql postgres -c "ALTER USER postgres PASSWORD 'postgres';"
    #sudo -u postgres psql postgres -c "DROP USER app_#{@APP_NAME};" > /dev/null
    #sudo -u postgres psql postgres -c "CREATE USER app_#{@APP_NAME} WITH PASSWORD 'app_#{@APP_NAME}' CREATEDB;"
    #we are using 'postgres' user for development

    # Allowing external hosts to connect to pgsql
    if ! grep -qe "0\.0\.0\.0\/0" $(sudo find /etc/postgresql/ -name pg_hba.conf); then
      echo " - Preparing 'pg_hba.conf'"
      sudo sed 's@127.0.0.1/32@0.0.0.0/0   @' $(sudo find /etc/postgresql/ -name pg_hba.conf) > ~/pg_hba.conf
      sudo mv ~/pg_hba.conf $(sudo find /etc/postgresql/ -name pg_hba.conf)
    fi

    # Defining pgsql to listen to external hosts
    if grep -qe "#listen_addresses" $(sudo find /etc/postgresql/ -name postgresql.conf); then
        echo " - Preparing 'postgresql.conf'"
        sudo sed -i "s/#listen_address.*/listen_addresses='*'/" $(sudo find /etc/postgresql/ -name postgresql.conf)
    fi

    # Restarting pgsql service
    sudo /etc/init.d/postgresql restart

  SHELL

  # =========== BOOT =========================
  config.vm.provision "shell", run: "always", inline: <<-SHELL

    # Initialize environment
    rm -rf /src/web-#{@APP_NAME}/tmp/pids/server.pid
    cd /src/web-#{@APP_NAME}
    if [ -f tmp/pids/server.pid ]; then
        kill $(cat tmp/pids/server.pid)
    fi
    bundle install
    rails server --binding=0.0.0.0 --daemon
  SHELL
end