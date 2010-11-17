A node.io job takes the following format

    var Job = require('node.io').Job;
    var options = {}, methods = {};
    exports.job = new Job(options, methods);

To run this job (e.g. saved as _myjob.js_) from the command line, run the following command in the same directory

    $ node.io myjob

Jobs typically contain an input, run, and output method. If omitted, input and output default to *STDIN* and *STDOUT*.

The full API is [available here](https://github.com/chriso/node.io/blob/master/docs/api.md).

## Getting started

The following very basic example highlights how to create and run a job.

_times2.js_
    
    var methods = {
        input: [0,1,2],
        run: function(num) {
            this.emit(num * 2);
        }
    };
    
    exports.job = new Job({}, methods);
    
To run _times2.js_, run the following command in the same directory

    $ node.io times2
        => 0\n2\n4\n
    
_times2.js_ can also be run inside another script

    var times2 = require('./times2'), nodeio = require('node.io');
    nodeio.start(times2, function(err) {
        //Called on completion. STDOUT => 0\n2\n4\n
    });
    
To capture the output, set the third parameter of `start` to true

    nodeio.start(times2, function(err, result) {
        //result = [0,2,4]
    }, true);
    
## Extending a job

A job's options and methods can be inherited and overridden using `job.extend(new_options, new_methods);`

_times4.js_

    var times2 = require('./times2');
    
    exports.job = times2.extend({}, {
        run: function(num) {
            this.emit(num * 4);
        }
    }
        
    // $ node.io times4   =>  0\n4\n8\n

## Input / output

Node.io can handle a variety of input / output situations.

To input an array
    
    input: [0,1,2,3,4]

To input a file line by line (auto-detects \n or \r\n)

    input: '/tmp/file.txt'

To read all files in a directory 

    input: '/tmp',
    
    run: function(path) {
        console.log(path); //Outputs the full path of each file/dir in /tmp
    }
    
    //To recurse subdirectories, set the 'recurse' option to true

To add your own input (e.g. from a database), use the following format

    input: function(start, num, callback) {
        //E.g. start=0, num=10 => first 10 rows 
        //     start=10, num=10 => second 10 rows
        //callback takes (err, input)
    }
    
To run the job indefinitely

    input: false
   
To output to a file

    output: '/tmp/output.txt'  //Defaults to \n for newline. Change by setting the 'newline' option
    
To add your own output, use the following format

    output: function(out) {
        //Note: out will be an array
    } 

Input / output can be optionally overridden at the command line

    $ node.io -i /tmp/input.txt -o /tmp/output.txt job
    
Node.io uses *STDIN* and *STDOUT* by default, so this is the same as calling

    $ cat /tmp/input.txt | node.io -s job > /tmp/output.txt
    
Note: the `-s` option at the command line omits any status or warnings messages being output
    
## Example 1 - resolve.js

The following job resolves a domain or list of domains.

_resolve.js_

    var Job = require('node.io').Job, dns = require('dns');
    
    var options = {
        max: 100,      //Run a maximum of 100 DNS requests concurrently
        timeout: 5,    //Timeout each request after 5s
        retries: 3     //Maximum of 3 retries before failing
    }
    
    var methods = {
        run: function(domain) {
            var self = this;
            
            dns.lookup(domain, 4, function(err, ip) {
                if (err) {
                    //The domain didn't resolve, retry
                    self.retry();
                } else {
                    //The domain resolved successfully
                    self.emit(domain + ',' + ip);
                }
            });
        },
        
        //fail() is called if the request times out or exceeds the maximum number of retries
        fail: function(status, domain) {
            this.emit(domain + ',failed');
        }   
    }
    
    exports.job = new Job(options, methods);
    
Try it out
    
    $ echo "google.com" | node.io resolve
        => google.com,66.102.11.104
    
..or with a list of domains

    $ cat domains.txt | node.io resolve
        
## Example 2 - coffee.js

The following job compiles all [CoffeeScript](http://jashkenas.github.com/coffee-script/) files in a directory and any subdirectories.

_coffee.js_

    var Job = require('node.io').Job, dns = require('dns');
    
    var options = {
        max: 10,         //Compile a max of 10 files concurrently
        recurse: true    //Compile .coffee scripts in subdirectories
    }
    
    var methods = {
        run: function(file) {
            var self = this, len = file.length;
            
            //Only compile .coffee files
            if (file.substr(len-7, len-1) === '.coffee') {
                this.exec('coffee -c "' + file + '"', function(err) {
                    if (err) {
                        self.exit(err);
                    } else {
                        self.finish();
                    }
                });
            } else {
                this.skip();
            }
        } 
    }
    
    exports.job = new Job(options, methods);
    
Try it out
    
    $ node.io -i /coffee/dir coffee
    
## Example 3 - reddit.js

The following job pulls the front page stories and scores from [reddit](http://reddit.com/) - just as a proof of concept, there are API's for this..

_reddit.js_

    var Job = require('node.io').Job;

    //Timeout after 10s, and only run the job once
    var options = {timeout:10, once:true};
    
    var methods = {
    
        input: false,
        
        run: function() {
            var self = this;
            
            this.getHtml('http://www.reddit.com/', function(err, $) {
                //Handle any http / parsing errors
                if (err) self.exit(err);
                
                var titles = [], scores = [], output = [];
                
                //Select all titles on the page
                $('a.title').each(function(a) {
                    titles.push(a.text);
                });
                
                //Select all scores on the page
                $('div.score.unvoted').each(function(div) {
                    scores.push(div.text);
                });
                
                //Mismatch? page probably didn't load properly
                if (scores.length != titles.length) {
                    self.exit('Title / score mismatch');
                }
                
                //Output = [score] title
                for (var i = 0, len = scores.length; i < len; i++) {
                    //Ignore upcoming stories
                    if (scores[i] == '&bull;') continue;
                    
                    //Check the data is ok
                    self.assert(scores[i]).isInt();
                    
                    output.push('['+scores[i]+'] '+titles[i]);
                }
                
                self.emit(output);
            });
        }
    }
    
    //Export the job
    exports.job = new Job(options, methods);
    
## Linking jobs together

Since node.io uses *STDIN* and *STDOUT* by default, jobs can be linked together. Be sure to add the `-s` option to intermediate `node.io` calls to prevent status/warning messages from being added to the output.

The following example uses _resolve.js_ from Example 1 and uses another job to filter out invalid domains before resolving.

_domains.txt_

    google.com
    youtube.com
    this*is^invalid.com

_valid_url.js_

    var Job = require('node.io').Job;
        
    var methods = {
        run: function(url) {
            this.assert(url).isUrl(); //If this fails, the url is not emitted
            this.emit(url);
        }
    }
    
    exports.job = new Job({}, methods);
    
To link the jobs

    $ cat domains.txt | node.io -s valid_url | node.io resolve 

## Distributing work

Node.io can currently partition the work among child processes to speed up execution, and will soon support distributing the work across multiple servers to create a work cluster.

**Note: Input and output is handled by the master process, while `job.run` is performed in parallel by each of the workers. Be careful of persistence outside of the [API](https://github.com/chriso/node.io/blob/master/docs/api.md).**

To enable this feature, either set the fork option to the number of workers you want to spawn, or use the `-f` command line option.

Enabling it in a job
    
    var options = {fork:4};
    
Or at the command line

    $ node.io -f 4 job
    
## Passing arguments to jobs

Any arguments after the job name on the command line are available in the job as a string

    $ node.io job arg1 arg2 arg3
    
    run: function() {
        console.log(this.options.args); //arg1 arg2 arg3
    }
    
## More examples

See [./examples](https://github.com/chriso/node.io/tree/master/examples/). Included examples are:

- `duplicates.js` - remove all duplicates in a list, or only output duplicate lines
- `validate.js` - filters a list with a variety of validation methods
- `resolve.js` - more robust version of the example above
- `word_count.js` - uses map reduce to count the occurrences of each word in a file
- `reddit.js` - web scraping example - pulls the front page stories and scores from [reddit](http://reddit.com/]
- `google_rank.js` - returns a domain's rank for a given keyword
- `google_pagerank.js` - find the pagerank of a URL
- `google_spell.js` - outputs the result of google suggest
- `coffee.js` - rescursively compile all .coffee files in a directory

See each file for usage details.