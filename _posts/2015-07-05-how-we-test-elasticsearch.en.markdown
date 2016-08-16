---
layout: post
title: How we test Elasticsearch
description: Writing tests for the elasticsearch engine in Ruby on Rails
date: 2015-07-05 00:00:00
---
One of our projects contains Elasticsearch as a fulltext search engine. It comes in handy for a few things such as an aggregating search, and a phonetic search that returns results when a user tries to find a misspelt word. Also Elasticsearch has a convenient `gem "elasticsearch-rails"`. It provides tools that solve some common problems and has a bunch of syntactic sugar. The installation process is not in the scope of this post, but you can install the gem following the instructions from the [official github repo](https://github.com/elastic/elasticsearch-rails#installation).


In this short article I want to share my experience of writing tests for Elasticsearch in our application. Before we start, it is important to highlight that we have only one model for the index (Trademark), but this code can be easily extended.


First of all, I was adding `Trademark.import(force: true, refresh: true)` to all examples that are related to Elasticsearch. It solved the problem associated with cleaning state between particular tests and refreshed the index manually. I have added this line to `before` block. Now let’s take a look at some sample unit test:

~~~ruby
describe SearchTrademarks do
  describe "#results" do
    let!(:trademark) do
      create(
        :trademark,
        serial_number: "213141"
      )
    end

    let(:search) { described_class.new(query: query).results }

    before do
      Trademark.import(force: true, refresh: true)

      create(:trademark, serial_number: "333")
    end

    context "when query matches with serial number" do
      let(:query) { "13" }

      it "returns particular trademark" do
        expect(search.first).to eq(trademark)
      end
    end
  end
end
~~~

Then my setup was working great until I realized that one problem is still there. Every time I had launched the test suite, It would reindex my test database and all data from the developing database got lost. When I returned to the development I had to reindex my development database again. It can become a bit tedious after a while.


To solve this I decided to use `gem "elasticsearch-extensions"`. It allowed me to programmatically start and stop an Elasticsearch cluster suitable for isolating tests. In order to avoid conflicts with development Elasticsearch cluster I used another port (by default it is 9200). It can be easily implemented by changing my `rails_helper.rb` to:

~~~ruby
# After other requires
require "rake"
require "elasticsearch/extensions/test/cluster/tasks"

config.before(:all) do
  if use_test_cluster?
    Elasticsearch::Model.client = Elasticsearch::Client.new(host: "localhost:9250")
    Elasticsearch::Extensions::Test::Cluster.start(port: 9250, nodes: 1)
  end
end

at_exit do
  if Elasticsearch::Extensions::Test::Cluster.running?(on: 9250)
    Elasticsearch::Extensions::Test::Cluster.stop(port: 9250)
  end
end

def use_test_cluster?
  !Elasticsearch::Extensions::Test::Cluster.running?(on: 9250)
end
~~~

I enjoyed the process of getting this problem resolved and pushing the code to [semaphoreci](https://semaphoreci.com/) (we use this service as a continuous integration tool). But suddenly the tests started failing! It was because semaphore knew nothing about the strange hostname called "localhost". To be honest, in CI we do not need the test cluster for Elasticsearch at all. The question is, how can we launch a test suite on semaphore without the test cluster and use it for the local environment at the same time?


My implementation is laughably simple. I have created an environment variable named "CI". If it exists I do not run the test cluster. Thus, we should reinforce conditional from `use_test_cluster?` by adding an extra clause:

~~~ruby
def use_test_cluster?
  !ENV["CI"] && !Elasticsearch::Extensions::Test::Cluster.running?(on: 9250)
end
~~~

That's it! Hope this will help you to figure out how to test Elasticsearch in your Ruby project.
