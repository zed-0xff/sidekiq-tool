# frozen_string_literal: true

require "bundler/gem_tasks"
require "rubocop/rake_task"

RuboCop::RakeTask.new

task default: %i[readme rubocop]

desc "update readme"
task :readme do
  data = []
  data << "# sidekiq-tool"
  data << ""
  data << "```"
  data << `./exe/sidekiq-tool`
  data << "```"

  File.write "README.md", data.join("\n")
end
