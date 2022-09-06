#!/usr/bin/env ruby
# frozen_string_literal: true
require 'json'
require 'optparse'
require 'redis'

@options = {
  url: ENV['REDIS_URL'],
  range: "0..-1",
  jid: [],
  job_class: [],
}

@commands = []

optparse = OptionParser.new do |opts|
  opts.banner = "Usage: #{File.basename($0)} [options]"

  opts.on("-u URL", "Redis URL (default: from REDIS_URL env var)") do |v|
    @options[:url] = v
  end

  opts.on("-q QUEUE", "--queue", "apply next commands to specified queue") do |v|
    @options[:queue] = v
  end

  opts.on("--jid JID", "(alias for --job-id)") do |v|
    @options[:jid] << v
  end
  opts.on("--job-id JID", "(can be used multiple times)") do |v|
    @options[:jid] << v
  end
  opts.on("--job-class CLASS", "(can be used multiple times)") do |v|
    @options[:job_class] << v
  end

  opts.separator("")

  opts.on("-l", "--list", "list queues (default)") do |v|
    @commands << [:list]
  end

  opts.on("-s", "--show [RANGE]",
          "show contents of queue",
          "see https://redis.io/commands/lrange/") do |v|
            @commands << [:show, v || "0..-1"]
          end

  opts.on("-S", "--schedule", "show scheduled jobs (respects queue parameter)") do |v|
    @commands << [:schedule]
  end
  opts.on("-P", "--processes", "show processes (respects queue parameter)") do |v|
    @commands << [:processes]
  end
  opts.on("-R", "--running-jobs", "show currently running jobs (respects queue/jid/job-class)") do |v|
    @commands << [:jobs]
  end

  opts.separator("")

  opts.on("--import-jobs", "add jobs from STDIN into queue") do
    @commands << [:import_jobs]
  end

  opts.on("--move-jobs [N]", Integer, "atomically move jobs to another queue") do |v|
    @commands << [:move_jobs, v]
  end
  opts.on("-Q QUEUE", "--dst-queue", "destination queue") do |v|
    @options[:dst_queue] = v
  end

  opts.separator("\nDestructive commands: (require confirmations)")

  opts.on("--delete-jobs [N]", Integer,
          "N limits number of jobs to delete, 0 (default) = delete all",
          "respects --job-id and --job-class parameters"
         ) do |v|
    @commands << [:delete_jobs, v]
  end
  opts.on("--export-jobs [N]", Integer, "same as delete, but job data is written to STDOUT beforehead") do |v|
    @commands << [:export_jobs, v]
  end
  opts.on("--delete-queue", "deletes ALL jobs from queue") do |v|
    @commands << [:delete_queue]
  end

  opts.separator("")

  opts.on("--confirm-delete-jobs", "jobs will not be deleted without this option") do |v|
    @options[:confirm_delete_jobs] = true
  end
  opts.on("--confirm-export-jobs") do |v|
    @options[:confirm_export_jobs] = true
  end
  opts.on("--confirm-queue-delete", "queue will not be deleted without this option") do |v|
    @options[:confirm_queue_delete] = true
  end

  opts.separator("")

  opts.on("-W", "--omit-weight", "Omit weight from schedule output (easier to parse)") do |v|
    @options[:omit_weight] = true
  end
  opts.on("-v", "--[no-]verbose", "Run verbosely") do |v|
    @options[:verbose] = v
  end
  opts.on("-k", "Bypass SSL verification (for debug/dev)") do |v|
    @options[:ssl_params] = { verify_mode: OpenSSL::SSL::VERIFY_NONE }
  end
end
optparse.parse!

if @commands.empty?
  puts optparse.help
  exit
end

@redis = Redis.new url: @options[:url], ssl_params: @options[:ssl_params]

def queue_name
  @options[:queue] || raise("no queue name set!")
end

def dst_queue_name
  @options[:dst_queue] || raise("no destination queue name set!")
end

def list
  @redis.keys("queue:*").each do |key|
    queue_name = key.sub(/^queue:/,'')
    puts queue_name
  end
end

def show range
  range = range.split("..").map(&:to_i)
  puts @redis.lrange("queue:#{queue_name}", *range).join("\n")
end

def delete_queue
  unless @options[:confirm_queue_delete]
    $stderr.puts "[!] cannot delete queue without confirmation"
    exit 1
  end
  @redis.del "queue:#{queue_name}"
end

def _process_jobs n
  if @options[:jid].empty? && @options[:job_class].empty? && n.to_i == 0
    $stderr.puts "[!] add N or --job-id or --job-class"
    exit 1
  end
  ndeleted = 0
  n = Float::INFINITY if n.nil? || n == 0
  # TODO: select in chunks
  @redis.lrange("queue:#{queue_name}", 0, -1).each do |jobdata|
    job = JSON.parse(jobdata)
    next if @options[:jid].any? && !@options[:jid].include?(job['jid'])
    next if @options[:job_class].any? && !@options[:job_class].include?(job['class'])
    ndeleted += @redis.lrem("queue:#{queue_name}", 1, jobdata).to_i
    yield jobdata if block_given?
    break if ndeleted >= n
  end
  ndeleted
end

def delete_jobs n
  unless @options[:confirm_delete_jobs]
    $stderr.puts "[!] cannot delete jobs without confirmation"
    exit 1
  end
  ndeleted = _process_jobs(n)
  puts "[=] deleted #{ndeleted} jobs"
end

def export_jobs n
  unless @options[:confirm_export_jobs]
    $stderr.puts "[!] cannot export jobs without confirmation"
    exit 1
  end
  _process_jobs(n) do |jobdata|
    puts jobdata
  end
end

def import_jobs
  n = 0
  while jobdata = gets
    jobdata.strip!
    raise "invalid job format: #{jobdata.inspect}" unless jobdata=~ /^\{.+\}$/
    @redis.rpush("queue:#{queue_name}", jobdata)
    n += 1
  end
  puts "[=] imported #{n} jobs"
end

def move_jobs n
  nmoved = _process_jobs(n) do |jobdata|
    puts jobdata if @options[:verbose]
    @redis.rpush("queue:#{dst_queue_name}", jobdata)
  end
  puts "[=] moved #{nmoved} jobs"
end

# can optionally accept queue/job_class/jid
def schedule
  match = []
  if @options[:queue]
    match << %Q|"queue":"#{@options[:queue]}"|
  end
  if @options[:job_class].size == 1
    match << %Q|"class":"#{@options[:job_class][0]}"|
  elsif @options[:job_class].size > 1
    raise "multiple job classes filter TBD"
  end
  if @options[:jid].size == 1
    match << %Q|"jid":"#{@options[:jid][0]}"|
  elsif @options[:jid].size > 1
    raise "multiple job ids filter TBD"
  end
  match = match.any? ? ("*"+match.join("*")+"*") : nil
  @redis.zscan_each("schedule", count: 1000, match: match).each do |sdata, weight|
    if @options[:omit_weight]
      puts sdata
    else
      puts "[#{sdata}, #{weight}]"
    end
  end
end

# slowest!
def processes
  @redis.smembers("processes").each do |pid|
    r = @redis.hgetall pid
    r['info'] = JSON.parse(r['info'])
    if @options[:queue]
      queues = r.dig('info', 'queues')
      next unless queues && queues.include?(@options[:queue])
    end
    r['workers'] = @redis.hgetall("#{pid}:workers")&.transform_values{ |x| JSON.parse(x) }
    puts r.to_json
    $stdout.flush # makes grepping faster
  end
end

# slow!
def jobs
  @redis.smembers("processes").each do |pid|
    workers = @redis.hgetall("#{pid}:workers")
    next unless workers
    workers.values.each do |wdata|
      wdata = JSON.parse(wdata)
      payload = JSON.parse(wdata['payload'])
      next if @options[:queue] && payload['queue'] != @options[:queue]
      next if @options[:jid].any? && !@options[:jid].include?(payload['jid'])
      next if @options[:job_class].any? && !@options[:job_class].include?(payload['class'])
      puts payload.to_json
      $stdout.flush # makes grepping faster
    end
  end
end

@commands.each do |cmd|
  send cmd[0], *cmd[1..-1]
end

