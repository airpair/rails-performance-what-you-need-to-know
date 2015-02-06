## 1 Introduction

I often hear that Rails is slow. This has become a common theme among the Ruby and Rails community. But it is actually a myth. It's easy to make your application up to 10x faster just by using Rails in the right way. Here's what you need to know to optimize your Rails application.

### 1.1 Steps for optimizing a Rails app

There are only two reasons why Rails application might be slow:

1.  Ruby and Rails are used where other tool might be a much better choice.
2.  Too much memory is consumed and hence garbage collection takes too long.

Rails is a nice framework, and Ruby is a simple and elegant language. But when misused, they hit performance hard. There are tasks that you'd better do with other tools. For example, databases are really good at big data processing, R language is better suited for statistics, and so on.

Memory is the #1 reason why any Ruby application is slow. The 80-20 rule of Rails performance optimization is: 80% of speedup comes from memory optimization, remaining 20% from everything else. Why memory consumption is important? Because the more memory you allocate, the more work Ruby GC (garbage collector) has to do. Rails already has a large memory footprint. Average app takes about 100M just after launch. If you are not careful with memory, it's possible that your process grows over 1G. With so much memory to collect, it's not unusual for GC to take 50% and more of application execution time.

## 2 How Can We Make A Rails App Faster?

There are three ways to make your app faster: scaling, caching, and code optimization.

Scaling is easy these days. Heroku basically does it for you, and Hirefire makes the process automatic. You can learn more about autoscaling [here](http://mikecoutermarsh.com/auto-scaling-heroku-with-hirefire/). Other hosted environments offer similar solutions. By all means, use them if you can. But keep in mind that scaling is not a silver performance bullet. If your app serves the single request in 5 minutes, no amount of scaling will help. Also, with Heroku + Hirefire it's almost too easy to break the bank. I've seen Hirefire scaling up one of my apps to 36 dynos and having me to pay a whopping $3100 for that. Sure thing, I went ahead, scaled down manually to 2 dynos, and optimized the code instead.

Rails caching is also easy to do. Rails fragment caching is very good in Rails 4. [Rails docs](http://guides.rubyonrails.org/caching_with_rails.html) is an excellent source of info on caching. Another good read is [Cheyne Wallace's article on Rails performance](http://www.cheynewallace.com/ruby-on-rails-performance-tuning/). It's also easy to setup Memcached these days. But as with scaling, caching is not an ultimate solution to performance problems. If your code is not working optimally, then you'll find yourself spending more and more resources on caching, to the point when caching stops making things faster.

The only reliable way to make your Rails app faster is code optimization. In Rails' case it's memory optimization. And, of course, if you follow my advice and avoid Rails to do things it's not designed for, you'll have even less code to optimize.

Let me tell you how to optimize your Rails app in 7 simple steps.

### 2.1 Avoid Memory Intensive Rails Features

Some features in Rails take more memory than necessary resulting in extra garbage collection runs. Here's the list.

#### 2.1.1 Serializers

Serializer is a convenient way to represent strings from the database in a Ruby data type.

<!-- code lang=ruby linenums=true -->

    class Smth < ActiveRecord::Base
      serialize :data, JSON
    end
    Smth.find(...).data
    Smth.find(...).data = { ... }
    But convenience comes with 3x memory overhead. If you store 100M in data column, expect to allocate 300M just to read it from the database.

It's more memory efficient to do the serialization yourself like this:

<!-- code lang=ruby linenums=true -->

    class Smth < ActiveRecord::Base
      def data
        JSON.parse(read_attribute(:data))
      end
      def data=(value)
        write_attribute(:data, value.to_json)
      end
    end

This will have only 2x memory overhead. Some people, including myself, have seen Rails' JSON serializers leak memory, about 10% of data size per request. I do not understand the reason behind this. Neither do I have a reproducible case. If you experienced that, or know how to reproduce the leak, please [let me know](https://www.airpair.com/book/adymo).

#### 2.1.2 Active Record

It's easy to manipulate the data with ActiveRecord. But ActiveRecord is essentially a wrapper on top of your data. If you have a 1G of data in the table, ActiveRecord representation of it will take 2G and, in some cases, more. Yes, in 90% of cases that overhead is justified by extra convenience that you get. But sometimes you don't need it.

One example where you can avoid ActiveRecord's overhead is bulk updates. The code below will neither instantiate any model nor run validations and callbacks.

<!-- code lang=ruby linenums=true -->

    Book.where('title LIKE ?', '%Rails%').update_all(author: 'David')

Behind the scenes it will just execute SQL UPDATE statement.

<!-- code lang=ruby linenums=true -->

    update books
      set author = 'David'
      where title LIKE '%Rails%'
    Another example is iteration over a large dataset. Sometimes you need only the data. No typecasting, no updates. This snippet just runs the query and avoids ActiveRecord altogether:
    result = ActiveRecord::Base.execute 'select * from books'
    result.each do |row|
      # do something with row.values_at('col1', 'col2')
    end

#### 2.1.3 String Callbacks

Rails callbacks like before/after save, before/after action and so on are heavily used. But the way you write them may kill your performance. Here are the 3 ways you can write, for example, before_save callback:

<!-- code lang=ruby linenums=true -->

    before_save :update_status
    before_save do |model|
      model.update_status
    end
    before_save "self.update_status"

First two ways are good to go, third is not. Why? Because to execute this callback Rails need to store execution context (variables, constants, global instances, etc.) that is present at the time of the callback. If your application is large, you end up copying a lot of data in the memory. Because callback can be executed any time, the memory cannot be reclaimed until your program finishes.

Having symbol callbacks saved me in one occasion 0.6 sec per request.

### 2.2 Write Less Ruby

This is my favorite step. As my university CS class professor liked to say, the best code is the one that doesn't exist. Sometimes the task at hand is better done with other tools. Most commonly it's a database. Why? Because Ruby is bad at processing large data sets. Like, very very bad. Remember, Ruby takes a large memory footprint. So, for example, to process 1G of data you might need 3G and more of memory. It will take several dozen seconds to garbage collect 3G. Good database can process the data in under a second. Let me show a few examples.

#### 2.2.1 Attribute Preloading

Sometimes after denormalization attributes of the model get stored in another database table. For example, imagine we're building a TODO list that consists of task. Each task can be tagged with one or several tags. Denormalized data model will look like this:

<!-- code lang=ruby linenums=true -->

    Tasks
     id
     name

    Tags
     id
     name

    Tasks_Tags
     tag_id
     task_id

To load tasks and their tags in Rails you would do this:

<!-- code lang=ruby linenums=true -->

    tasks = Task.find(:all, :include => :tags)
        > 0.058 sec

The problem with the code is that it creates an object for every Tag. That takes memory. Alternative solution is to do tag preloading in the database.

<!-- code lang=ruby linenums=true -->

    tasks = Task.select <<-END
          *,
          array(
            select tags.name from tags inner join tasks_tags on (tags.id = tasks_tags.tag_id)
            where tasks_tags.task_id=tasks.id
          ) as tag_names
        END
        > 0.018 sec

This requires memory only to store an additional column that has an array of tags. No wonder it's 3x faster.

#### 2.2.2 Data Aggregation

By data aggregation I mean any code that to summarizes or analyzes datasets. These operations can be simple sums, or anything more complex. Let's take group rank for example. Let's assume we have a dataset with employees, departments and salaries and we want to calculate the employee's rank within a department by salary.

<!-- code lang=ruby linenums=true -->

    SELECT * FROM empsalary;

      depname  | empno | salary
    -----------+-------+-------
     develop   |     6 |   6000
     develop   |     7 |   4500
     develop   |     5 |   4200
     personnel |     2 |   3900
     personnel |     4 |   3500
     sales     |     1 |   5000
     sales     |     3 |   4800
    

You can calculate the rank in Ruby:

<!-- code lang=ruby linenums=true -->

    salaries = Empsalary.all
    salaries.sort_by! { |s| [s.depname, s.salary] }
    key, counter = nil, nil
    salaries.each do |s|
     if s.depname != key
      key, counter = s.depname, 0
     end
     counter += 1
     s.rank = counter
    end

With 100k records in empsalary table this program finishes in 4.02 seconds. Alternative Postgres query using window functions does the same job more than 4 times faster in 1.1 seconds.

<!-- code lang=ruby linenums=true -->

    SELECT depname, empno, salary, rank()
    OVER (PARTITION BY depname ORDER BY salary DESC)
    FROM empsalary;

      depname  | empno | salary | rank 
    -----------+-------+--------+------
     develop   |     6 |   6000 |    1
     develop   |     7 |   4500 |    2
     develop   |     5 |   4200 |    3
     personnel |     2 |   3900 |    1
     personnel |     4 |   3500 |    2
     sales     |     1 |   5000 |    1
     sales     |     3 |   4800 |    2

4x speedup is already impressive, but sometimes you get more, up to 20x. One example from my own experience. I had a 3-dimensional OLAP cube with 600k data rows. My program did slicing and aggregation. When done in Ruby, it took 1G of memory and finished in about 90 seconds. Equivalent SQL query finished in 5 seconds.

### 2.3 Fine-tune Unicorn

Chances are you are already using Unicorn. It's the fastest web server that you can use with Rails. But you can make it even faster.

#### 2.3.1 App Preloading

Unicorn lets you preload your Rails application before forking worker processes. This has two advantages. First, with copy-on-write friendly GC (Ruby 2.0 and later) workers can share data loaded into memory by master process. That data will be transparently copied by an operating system in case worker changes it. Second, preloading reduces worker startup time. It's normal for Rails workers to be restarted (we'll talk about this in a moment), so the faster they restart, the better performance you get.

To turn on application preloading, simply include this line into your unicorn configuration file:

<!-- code lang=ruby linenums=true -->

    preload_app true

#### 2.3.2 GC Between Requests

Remember, GC amounts for up to 50% of your application runtime. That's not the only problem. GC is also unpredictable and happens exactly when you don't want it. What can we do about that?

First thing that comes to mind, what if we disable GC altogether? It turns out that's a bad idea. Your application can easily take 1G of memory before you notice. If you run several workers on your server, you will run out of memory even on a self-hosted hardware. Not to mention Heroku with its default 512M limit.

There's a better idea. If we cannot avoid GC, we can make it more predictable and run when there's nothing else to do. For example, between requests. It's actually easy to do this with Unicorn.

For Ruby < 2.1 there's OobGC Unicorn module:

<!-- code lang=ruby linenums=true -->

    require 'unicorn/oob_gc'
        use(Unicorn::OobGC, 1)   # "1" here means "force GC after every 1 request"
    For Ruby >= 2.1 it's better to use [gctools](https://github.com/tmm1/gctools):

    require 'gctools/oobgc'
    use(GC::OOB::UnicornMiddleware)

GC between requests has its caveats. Most importantly, this improves only perceived application performance. Meaning that the users will definitely see the optimization. You server hardware will actually have to do more work. Instead of doing GC as necessary, you will force your server to do it more often. So make sure you have enough resources to run GC and enough workers to serve incoming requests while other workers are busy with GC.

### 2.4 Limit Growth

I gave you several examples already where your application can grow up to 1G of memory. Taking a huge piece of memory by itself might not be a problem if you have a lot of memory. The real problem is that Ruby might not give that memory back to the operating system. Let me explain why.

Ruby allocates memory in two heaps. All Ruby objects go to Ruby's own heap. Each object has 40 bytes (on a 64-bit system) to store its data. When object needs to store more, it will allocate space in operating system's heap. When object is garbage collected and then freed, the space in the operating system's heap goes back to the operating system of course. But the space reserved for the object itself in Ruby heap is simply marked as free.

This means that Ruby heap can only grow. Imagine, you read 1 million rows, 10 columns in each row from the database. For that you allocate at least 10 million objects to store the data. Usually, Rails workers in average applications take 100M after start. To accommodate the data, the worker will grow by additional 400M (10 million objects, 40 bytes each). Even after you done with the data and garbage collect it, the worker will stay at 500M.

Disclaimer, Ruby GC does have the code to shrink the heap. I have yet to see that happen in reality because the conditions for heap to shrink rarely happen in production applications.

If your workers can only grow, the obvious solution is to restart the ones that get too big. Some hosting services, Heroku for example, do that for you. Let's see the other ways to do that.

#### 2.4.1 Internal Memory Control

Trust in God, but lock your car. There're 2 types of memory limits your application can control on its own. I call them kind and hard.

**Kind** memory limit is the one that's enforced after each request. If worker is too big, it can quit and Unicorn master will start the new one. That's why I call it "kind", it doesn't abrupt your application.

To get the process memory size, use RSS metric on Linux and MacOS or [OS gem](https://github.com/rdp/os) on Windows. Let me show how to implement this limit in Unicorn configuration file:

<!-- code lang=ruby linenums=true -->

    class Unicorn::HttpServer
     KIND_MEMORY_LIMIT_RSS = 150 #MB

     alias process_client_orig process_client
     undef_method :process_client
     def process_client(client)
      process_client_orig(client)
      rss = `ps -o rss= -p #{Process.pid}`.chomp.to_i / 1024
      exit if rss > KIND_MEMORY_LIMIT_RSS
     end
    end

**Hard** memory limit is set by asking the operating system to kill your worker process if it grows too much. On Unix you can call setrlimit to set the RSS limit. To my knowledge, this only works on Linux. MacOS implementation was broken at some point. I'd appreciate any new information on that matter.

This the snippet from Unicorn configuration file with the hard limit:

<!-- code lang=ruby linenums=true -->

    after_fork do |server, worker|
      worker.set_memory_limits
    end

    class Unicorn::Worker

      HARD_MEMORY_LIMIT_RSS = 600 #MB
      def set_memory_limits
        Process.setrlimit(Process::RLIMIT_AS, HARD_MEMORY_LIMIT * 1024 * 1024)
      end

    end

#### 2.4.2 External Memory Control

Self-control does not save you from occasional OOM (out of memory). Usually you should setup some external tools. On Heroku there's no need as they have their own monitoring. But if you're self-hosting, it's a good idea to use [monit](http://mmonit.com/monit/), [god](https://github.com/mojombo/god), or any other monitoring solution.

### 2.5 Tune Ruby GC

In some cases you can tune Ruby GC to improve its performance. I'd say that these GC tuning is becoming less and less important, as the default settings in Ruby 2.1 and later are already good for most people.

To fine-tune GC you need to know how it works. This is a separate topic that does not belong to this article. To learn more, read a thorough [Demystifying the Ruby GC](http://samsaffron.com/archive/2013/11/22/demystifying-the-ruby-gc) article by Sam Saffron. In my upcoming Ruby Performance book I dig into even deeper details of Ruby GC. [Subscribe here](http://ruby-performance-book.com/) and I will send you email when I finish at least the beta version of the book.

My best advice is, probably, _do not change GC settings_ unless you know what you are doing and have a good theory on how that can improve performance. This is especially true for Ruby 2.1 and later users.

I know only of one case when GC tuning helps. It's when you load large amounts of data in a single go. Here's when you can decrease the frequency of GC runs by changing the following environment variables: RUBY_GC_HEAP_GROWTH_FACTOR, RUBY_GC_MALLOC_LIMIT, RUBY_GC_MALLOC_LIMIT_MAX, RUBY_GC_OLDMALLOC_LIMIT, and RUBY_GC_OLDMALLOC_LIMIT.

Note, these are variables from Ruby 2.1 and later. Previous versions have less variables, and often use different names.

RUBY_GC_HEAP_GROWTH_FACTOR, default value 1.8, controls how much the Ruby heap grows when there's not enough heap space to accommodate new allocations. When you work with large amounts of objects, you want your heap space grow faster. In this case, increase the heap growth factor.

Memory limits define how often GC is triggered when you allocate memory on operating system's heap. Default limit values in Ruby 2.1 and later are:

<!-- code lang=ruby linenums=true -->

    New generation malloc limit RUBY_GC_MALLOC_LIMIT 16M
    Maximum new generation malloc limit RUBY_GC_MALLOC_LIMIT_MAX 32M
    Old generation malloc limit RUBY_GC_OLDMALLOC_LIMIT 16M
    Maximum old generation malloc limit RUBY_GC_OLDMALLOC_LIMIT_MAX 128M

Let me briefly explain what these values mean. With the configuration as above, for every 16M to 32M allocated by new objects, and for every 16M to 128M allocated by old objects ("old" means that an object survived at least 1 garbage collection call) Ruby will run GC. Ruby dynamically adjust the current limit value depending on your memory allocation pattern .

So, if you have small number of objects that take large amounts of memory (for example, read large files into one string), you can increase the limits to trigger GC less often. Remember to increase all 4 limits, better proportionally to their default values.

My advice, however, differs from what other people recommend. What worked for me might not work for you. Here are the articles describing what worked for [Twitter](http://blog.evanweaver.com/2009/04/09/ruby-gc-tuning/) and for [Discourse](https://meta.discourse.org/t/tuning-ruby-and-rails-for-discourse/4126).

### 2.6 Profile

Sometimes none of the ready-to-use advice can help you, and you need to figure out what's wrong yourself. That's when you use the profiler. [Ruby-Prof](https://github.com/ruby-prof/ruby-prof) is what everybody uses in Ruby world.

To learn more on profiling, read [Chris Heald's](https://www.coffeepowered.net/2013/08/02/ruby-prof-for-rails/) and [my own](http://www.acunote.com/blog/2008/02/make-your-ruby-rails-applications-fast-performance-and-memory-profiling.html) articles on using ruby-prof with Rails. Later also has, albeit outdated, advice on memory profiling.

### 2.7 Write Performance Tests

The last, but by all means not the least important step in making Rails faster is to make sure that slowdowns do not happen again after you fix them. Rails 3.x has [performance testing and profiling framework](http://guides.rubyonrails.org/v3.2.13/performance_testing.html) bundled in. For Rails 4 you can use the same framework extracted into [rails-perftest gem](https://github.com/rails/rails-perftest).

## 3 Closing Thoughts

It's not possible to cover everything on Ruby and Rails performance optimization in one post. So lately I decided to summarize my knowledge in a book. If you find my advice helpful, please [sign up for the mailinglist](http://ruby-performance-book.com/) and I'll notify you once I have an early preview of this book ready. Now, go ahead, and make your Rails application faster!