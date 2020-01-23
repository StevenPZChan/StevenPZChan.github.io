### Setup in Ubuntu
* Install Ruby RubyGems
```shell
# sudo apt install -y ruby-full rubygems
# sudo snap install ruby --classic
gpg --keyserver hkp://pgp.mit.edu --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
curl -sSL https://get.rvm.io | bash -s stable --rails
gem sources --add https://mirrors.tuna.tsinghua.edu.cn/rubygems/ --remove https://rubygems.org/
sudo gem update --system
```
* Install Node
```shell
sudo apt install -y nodejs
```
* Install python2
```shell
sudo apt install -y python
```
* Install jekyll
```shell
sudo gem install jekyll
jekyll --version
```
* Bundle build
```shell
bundle config mirror.https://rubygems.org https://mirrors.tuna.tsinghua.edu.cn/rubygems
bundle install
```

