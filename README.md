# README

This README would normally document whatever steps are necessary to get the
application up and running.

Things you may want to cover:

* Ruby version - ruby 2.4.2p198

* System dependencies

* Configuration

* Database creation

* Database initialization

* How to run the test suite

* Services (job queues, cache servers, search engines, etc.)

* Deployment instructions

* ...

In this article you will learn how to integrate ElasticSearch into a Rails application.


A full-text search engine examines all of the words in every stored document as it tries to match search criteria (text specified by a user) wikipedia. For example, if you want to find articles that talk about Rails, you might search using the term “rails”. If you don’t have a special indexing technique, it means fully scanning all records to find matches, which will be extremely inefficient. One way to solve this is an “inverted index” that maps the words in the content of all records to its location in the database.

Create the Rails App
Type the following at the command prompt:

$ rails new blog
$ cd blog
$ bundle install
$ rails s

Create the articles controller using the Rails generator, add routes to config/routes.rb, and add methods for showing, creating, and listing articles.

$ rails g controller articles
Then open config/routes.rb and add this resource:

Blog::Application.routes.draw do
  resources :articles
end
Now, open app/controllers/articles_controller.rb and add methods to create, view, and list articles.

def index
  @articles = Article.all
end

def show
  @article = Article.find params[:id]
end

def new
end

def create
  @article = Article.new article_params
  if @article.save
    redirect_to @article
  else
    render 'new'
  end
end

private
  def article_params
    params.require(:article).permit :title, :text
  end
Article Model
We’ll need a model for the articles, so generate it like so:

$ rails g model Article title:string text:text
$ rake db:migrate
Views
New Article Form
Create a new file at app/views/articles/new.html.erb with the following content:

<h1>New Article</h1>  

<%= form_for :article, url: articles_path do |f| %>

  <% if not @article.nil? and @article.errors.any? %>
  <div id="error_explanation">
    <h2><%= pluralize(@article.errors.count, "error") %> prohibited
      this article from being saved:</h2>
    <ul>
    <% @article.errors.full_messages.each do |msg| %>
      <li><%= msg %></li>
    <% end %>
    </ul>
  </div>
  <% end %>

  <p>
    <%= f.label :title %><br>
    <%= f.text_field :title %>
  </p>

  <p>
    <%= f.label :text %><br>
    <%= f.text_area :text %>
  </p>

  <p>
    <%= f.submit %>
  </p>
<% end %>

<%= link_to '<- Back', articles_path %>
Show One Article
Create another file at app/views/articles/show.html.erb:

<p>
  <strong>Title:</strong>
  <%= @article.title %>
</p>

<p>
  <strong>Text:</strong>
  <%= @article.text %>
</p>

<%= link_to '<- Back', articles_path %>
List All Articles
Create a third file at app/views/articles/index.html.erb:

<h1>Articles</h1>

<ul>
  <% @articles.each do |article| %>
    <li>
      <h3>
        <%= article.title %>
      </h3>
      <p>
        <%= article.text %>
      </p>
    </li>
  <% end -%>
</ul>
<%= link_to 'New Article', new_article_path %>
You can now add and view articles. Make sure you start the Rails server and go to http://localhost:3000/articles. Click on “New Article” and add a few articles. These will be used to test our full-text search capabilities.

Integrate ElasticSearch
Currently, we can find an article by id only. Integrating ElasticSearch will allow finding articles by any word in its title or text.

Install for Ubuntu and Mac
Ubuntu
Go to elasticsearch.org/download and download the DEB file. Once the file is local, type:

$ sudo dpkg -i elasticsearch-[version].deb
Mac
If you’re on a Mac, Homebrew makes it easy:

$ brew install elasticsearch
Validate Installation
Open this url: http://localhost:9200 and you’ll see ElasticSearch respond like so:

{
  "status" : 200,
  "name" : "Anvil",
  "version" : {
    "number" : "1.2.1",
    "build_hash" : "6c95b759f9e7ef0f8e17f77d850da43ce8a4b364",
    "build_timestamp" : "2014-06-03T15:02:52Z",
    "build_snapshot" : false,
    "lucene_version" : "4.8"
  },
  "tagline" : "You Know, for Search"
}
Add Basic Search
Create a controller called search, along with a view so you can do something like: /search?q=ruby.

Gemfile
gem 'elasticsearch-model'
gem 'elasticsearch-rails'
Remember to run bundle install to install these gems.

Search Controller
Create The SearchController:

$ rails g controller search
Add this method to app/controller/search_controller.rb:

def search
  if params[:q].nil?
    @articles = []
  else
    @articles = Article.search params[:q]
  end
end
Integrate Search into Article
To add the ElasticSearch integration to the Article model, require elasticsearch/model and include the main module in Article class.

Modify app/models/article.rb:

require 'elasticsearch/model'

class Article < ActiveRecord::Base
  include Elasticsearch::Model
  include Elasticsearch::Model::Callbacks
end
Article.import # for auto sync model with elastic search
Search View
Create a new file at app/views/search/search.html.erb:

<h1>Articles Search</h1>

<%= form_for search_path, method: :get do |f| %>
  <p>
    <%= f.label "Search for" %>
    <%= text_field_tag :q, params[:q] %>
    <%= submit_tag "Go", name: nil %>
  </p>
<% end %>

<ul>
  <% @articles.each do |article| %>
    <li>
      <h3>
        <%= link_to article.title, controller: "articles", action: "show", id: article._id%>
      </h3>
    </li>
  <% end %>
</ul>
Search Route
Add the search route to _config/routes.rb-:

get 'search', to: 'search#search'
