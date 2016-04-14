---
published: true
title: Grunt, motherfucker! Do you need it?
layout: post
categories: grunt gulp nodejs node.js npm javascript
---

Lately Node.js task runners have been really kicking up a storm. Grunt, Gulp, and Broccoli are all providing slightly different approaches to performing similar tasks.

For certain tasks these task runners awesome, but for others they're unnecessary.

Many Node.js projects/modules are simple affairs that require running some tests, code quality checks and...yep, that's it. You could accomplish these tasks using any of the previously referenced task runners, or you could just use the CLI tool equivalent.

Personally I use CLI tools via Make 90% of the time. Why? It's faster, more reliable, doesn't require installing extra node_modules into my project (unless I want to install specific local versions), and doesn't require knowledge of how these plugins are configured.

For example, here's a Gruntfile that runs code quality checks using JSHint, linelint and lintspaces, tests using Mocha and finally builds a bundle using Browserify.

```javascript

'use strict';

module.exports = function (grunt) {

	grunt.initConfig({
		pkg: grunt.file.readJSON('package.json'),

		lintspaces: {
			all: {
				src: ['./src/**/*.js'],
				options: {
					newline: true,
					trailingspaces: true,
					indentation: 'spaces',
          spaces: 2
				}
			}
		},

		jshint: {
			with_overrides: {
				jshintrc: true,
				files: {
					src: ['./src/**/*.js']
				}
			}
	  },

	  mocha: {
		  test: {
		  	options: {
		      reporter: 'spec'
		    },
		    src: ['tests/*.js'],
		  },
		},

		browserify: {
			dist: {
		    options: {
		      browserifyOptions: {
		      	'e': './lib/LoggerFactory.js',
		      	'o': './dist/fhlog.js',
		      	's': 'fhlog'
		      }
		    }
		  }
		}
	});

	grunt.loadNpmTasks('grunt-mocha');
	grunt.loadNpmTasks('grunt-browserify');
	grunt.loadNpmTasks('grunt-lintspaces');
	grunt.loadNpmTasks('grunt-contrib-jshint');

	grunt.registerTask('format', ['lintspaces:all', 'jhint:with_overrides']);
	grunt.registerTask('build', ['format', 'browserify:dist']);
	grunt.registerTask('test', ['build', 'mocha:test']);
};

```

Here's the equivalent Makefile.

```bash
srcFiles = $(shell find ./lib -type f -name '*.js' | xargs)

.PHONY : test

default: format

# Run tests, then build the JavaScript bundle
build:format
	$(browserify) -s fhlog -e ./lib/LoggerFactory.js -o ./dist/fhlog.js
	@echo "Build succeeded!\n"

# Test files for formatting and errors, then run tests
test:build
	$(mocha) -R spec ./test/*.js

# Test file formatting and for errors
format:
	$(linelint) $(srcFiles) $(testFiles)
	@echo "linelint pass!\n"
	$(lintspaces) -nt -i js-comments -d spaces -s 2 $(srcFiles)
	@echo "lintspaces pass!\n"
	$(jshint) $(srcFiles)
	@echo "JSHint pass!\n"

```

The Makefile is more concise but this probably isn't a huge deal to most people. The reason I like make is due to it's predictable nature. Certain Grunt plugins will continue to run despite encountering errors, which is something you generally wouldn't want to occur. I've also been burned by assuming all plugins support targets (yeah, turns out they don't!) and plugins not applying options correctly to the underlying tool.

On the other hand Grunt is fantastic for handling concatenation of CSS, performing tasks upon changes to files, and for performing more complex tasks that would be difficult via CLI tools. This is particularly true for client-side projects.

Grunt is great for larger projects, but for many Node.js projects it's unnecessary. Choose the best tool for the job.

![](https://dl.dropboxusercontent.com/u/4401092/blog/images/2015/06/samuel-1.jpg)
