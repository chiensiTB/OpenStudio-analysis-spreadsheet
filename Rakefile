require 'bundler'
Bundler.setup

require 'rake'
require 'rake/clean'

require 'openstudio-aws'
require 'openstudio-analysis'
require 'colored'
require 'pp'

require 'rubygems'
require 'zip'

CLEAN.include('*.pem', '*.pub', './projects/*.json', '*.json', 'faraday.log')

# Command-line arguments in Rake: http://viget.com/extend/protip-passing-parameters-to-your-rake-tasks
def get_project(excel_file = '')
  # If excel_file is not pre-specified, request it as input
  unless excel_file && !excel_file.empty?
    # Determine the project file to run.  This will list out all the xlsx files and give you a
    # choice from which to choose
    puts
    puts 'Select which project to run from the list below:'.cyan.underline
    puts 'Note: if this list is too long, simply remove xlsx files from the ./projects directory'.cyan
    projects = Dir['./projects/*.{xlsx,csv}'].reject { |i| i =~ /~\$.*/ }
    projects.each_index do |i|
      puts "  #{i + 1}) #{File.basename(projects[i])}".green
    end
    puts
    print "Selection (1-#{projects.size}): ".cyan
    n = $stdin.gets.chomp
    n_i = n.to_i
    if n_i == 0 || n_i > projects.size
      puts "Could not process your selection. You entered '#{n}'".red
      exit
    end

    excel_file = projects[n_i - 1]
  end

  # Open the file
  analysis = nil
  run_options = nil
  if excel_file && File.exist?(excel_file)
    if File.extname(excel_file) == '.xlsx'
      excel = OpenStudio::Analysis::Translator::Excel.new(excel_file)
      excel.process

      # Ideally there would be no difference between excel and the csv,
      # but as you can tell, there is. ugh-ly.
      analysis = excel.analysis
      run_options = excel.settings
      run_options.delete('aws_tag')
      run_options['aws_tags'] = excel.aws_tags
      run_options.merge!(excel.run_setup)
      run_options['analysis_type'] = excel.problem['analysis_type']

    elsif File.extname(excel_file) == '.csv'
      csv = OpenStudio::Analysis::Translator::Datapoints.new(excel_file)
      csv.process

      analysis = csv.analysis
      run_options = {}
      run_options['cluster_name'] = csv.cluster_name

      # AWS settings
      run_options['openstudio_server_version'] = csv.settings[:os_server_version]
      run_options['server_instance_type'] = csv.run_setup[:server_instance_type]
      run_options['worker_instance_type'] = csv.run_setup[:worker_instance_type]
      run_options['worker_nodes'] = csv.settings[:worker_node_number]
      run_options['aws_tags'] = csv.settings[:aws_tags]
      run_options['user_id'] = csv.settings[:user_id]
      run_options['proxy_port'] = csv.settings[:proxy_port]

      run_options['run_data_point_filename'] = 'run_openstudio_workflow_monthly.rb'
      run_options['simulate_data_point_filename'] = 'simulate_data_point.rb'
      run_options['analysis_type'] = csv.settings[:analysis_type]
    end
  else
    puts "Could not find input excel file: #{excel_file}".red
    exit 1
  end

  [analysis, run_options]
end

def create_cluster(options)
  # Verify that all the data are needed to create the cluster
  fail 'No cluster name in options' if options['cluster_name'].nil?
  fail 'No worker node count defined in options' if options['worker_nodes'].nil?
  fail 'No OpenStudio server version defined in options' if options['openstudio_server_version'].nil?
  fail 'No user id defined in options' if options['user_id'].nil?
  fail 'No server instance type defined in options' if options['server_instance_type'].nil?
  fail 'No worker instance type defined in options' if options['worker_instance_type'].nil?
  fail 'No worker instance type defined in options' if options['aws_tags'].nil?

  if File.exist?("#{options['cluster_name']}.json")
    puts
    puts "It appears that a cluster for #{options['cluster_name']} is already running. \
If this is not the case then delete ./#{ options['cluster_name']}.json file. \
Or run `rake clean`".red
    puts 'Will try to continue'.cyan
  else
    puts "Creating cluster for #{options['cluster_name']}".cyan
    puts 'Validating cluster options'.cyan

    if options['worker_nodes'].to_i.to_i == 0
      puts 'Number of workers set to zero... will continue'.cyan
    end

    puts "Number of worker nodes set to #{options['worker_nodes'].to_i}".cyan
    puts 'Starting cluster...'.cyan

    # Don't use the old API (Version 1)
    ami_version = options['openstudio_server_version'][0] == '2' ? 3 : 2
    aws_options = {
        ami_lookup_version: ami_version,
        openstudio_server_version: options['openstudio_server_version']
    }
    aws = OpenStudio::Aws::Aws.new(aws_options)

    server_options = {
        instance_type: options['server_instance_type'],
        user_id: options['user_id'],
        tags: options['aws_tags']
    }

    worker_options = {
        instance_type: options['worker_instance_type'],
        user_id: options['user_id'],
        tags: options['aws_tags']
    }

    start_time = Time.now

    # Create the server & worker
    aws.create_server(server_options)
    aws.save_cluster_info "#{options['cluster_name']}.json"
    aws.print_connection_info

    aws.create_workers(options['worker_nodes'].to_i, worker_options)
    aws.save_cluster_info "#{options['cluster_name']}.json"
    aws.print_connection_info

    server_dns = "http://#{aws.os_aws.server.data.dns}"

    puts "Cluster setup in #{(Time.now - start_time).round} seconds. Awaiting analyses.".cyan
    puts "Server IP address #{server_dns}".cyan
  end
end

def configure_target_server(cluster_name, target)
  # Choose target server and return the DNS
  server_dns = nil
  case target.downcase
    when 'vagrant'
      server_dns = 'http://192.168.99.100:8080'
    when 'nrel24a'
      server_dns = 'http://bball-130553.nrel.gov:8080'
    when 'nrel24b'
      server_dns = 'http://bball-130590.nrel.gov:8080'
    when 'nrel24'
      server_dns = 'http://bball-130449.nrel.gov:8080'
    when 'local_development'
      server_dns = 'http://localhost:3000'
    when 'local'
      server_dns = 'http://localhost:3000'
    when 'aws'
      if File.exist?("#{cluster_name}.json")
        json = JSON.parse(File.read("#{cluster_name}.json"), symbolize_names: true)
        server_dns = "http://#{json[:server][:dns]}"
      end
    else
      server_dns = target
  end
end

def unzip_archive(archive, dest)
  # Unzip an archive to a destination directory using Rubyzip gem

  # Adapted from examples at...
  #  https://github.com/rubyzip/rubyzip
  #  http://seenuvasan.wordpress.com/2010/09/21/unzip-files-using-ruby/
  Zip::File.open(archive) do |zf|
    zf.each do |f|
      f_path = File.join(dest, f.name)
      FileUtils.mkdir_p(File.dirname(f_path))
      zf.extract(f, f_path) unless File.exist?(f_path) # No overwrite
    end
  end
end

def run_analysis(analysis, run_options, target = 'aws', download = false)
  puts 'Running the analysis'

  server_dns = configure_target_server(run_options['cluster_name'], target)

  unless server_dns
    puts "There doesn't appear to be a cluster running for this project #{run_options['cluster_name']}"
    exit 1
  end

  formulation_file = "./analysis/#{File.basename(analysis.name)}.json"
  analysis_zip_file = "./analysis/#{File.basename(analysis.name)}.zip"

  # Project data
  options = {hostname: server_dns}
  api = OpenStudio::Analysis::ServerApi.new(options)

  begin
    if run_options[:batch_run_method]
      analysis_id = api.run(formulation_file,
                            analysis_zip_file,
                            run_options['analysis_type'],
                            { batch_run_method: run_options[:batch_run_method]}
      )
    else
      analysis_id = api.run(formulation_file,
                            analysis_zip_file,
                            run_options['analysis_type']
      )
    end
  rescue => e
    puts "Analysis submission failed with message `#{e.message}` in:\n#{e.backtrace.join("\n")}"
  end


  # Report some useful info
  puts
  puts "Analysis type is: #{run_options['analysis_type']}".bold.cyan
  puts "Server URL is: #{server_dns}".bold.cyan

  # If download option selected:
  # a. Monitor for completiong
  # b. Download results (R data frame and data point .zips) to ./results/#{analysis_id}
  # c. Clean up by deleting the analysis on the server
  if download
    puts
    puts 'Waiting to download analysis results... '.cyan

    # These are hard coded for now...
    check_interval = 15 # sec
    max_time = 3600 # sec

    Timeout.timeout(max_time) do
      begin
        # Monitor the server and wait for it to respond
        loop do
          print '.'
          status = api.get_analysis_status(analysis_id, 'batch_run')
          if status && status == 'completed'
            puts 'analysis completed!'

            out_dir = "./results/#{analysis_id}"
            FileUtils.mkdir_p(out_dir)
            puts "Download directory is: #{out_dir}"

            # Download R data frame
            puts
            puts 'Downloading R data frame...'.cyan
            ok, f = api.download_dataframe(analysis_id, out_dir)
            if ok
              puts 'Downloaded R data frame succesfully.'
            else
              puts 'Error downloading R data frame.'
            end

            # Download all the datapoints
            data_points = api.get_datapoint_status(analysis_id, 'completed')
            puts
            puts 'Downloading all data points...'.cyan
            if data_points.nil? || data_points.empty?
              puts "\tNo completed data points found even though the analysis completed!"

            else
              data_points.each do |dp|
                if dp[:final_message] == 'completed normal'
                  puts "\tDownloading data point #{dp[:_id]}"
                  ok, f = api.download_datapoint(dp[:_id], out_dir)

                  if ok
                    puts "\tExtracting data point #{dp[:_id]}"
                    dest = File.join(File.dirname(f), File.basename(f, '.zip'))
                    unzip_archive(f, dest)
                    File.delete(f)
                  else
                    puts "\tError downloading data point #{dp[:_id]}"
                  end

                else
                  puts "\tError found in data point #{dp[:_id]}"
                end
              end
            end

            # Clean up project
            puts
            puts 'Cleaning up...'.cyan
            api.delete_project(project_id)

            break
          end

          sleep check_interval
        end

          # On timeout...
      rescue TimeoutError => e
        puts 'Time expired before analysis completed! Download aborted.'.bold.red
      end
    end
  end

  # Final stuff
  if target.downcase == 'aws'
    puts
    puts 'Make sure to check the AWS console (N. Virginia Region) and terminate any OpenStudio instances when you are finished!'.bold.red
  end
end

# Run multiple analyses using the models in the Excel file. This needs to be
# deprecated.
def run_analyses(excel, target = 'aws', download = false)
  puts 'Running the analysis'

  # Which server?
  server_dns = configure_target_server(excel, target)

  unless server_dns
    puts "There doesn't appear to be a cluster running for this project #{excel.cluster_name}"
    exit 1
  end

  # for each model in the excel file submit the analysis
  excel.models.each do |model|
    # parse the file and check if the instance appears to be up

    file_name = nil
    if excel.models.size > 1
      file_name = "#{excel.name.snake_case} #{model[:name]}".snake_case
    else
      file_name = "#{excel.name.snake_case}".snake_case
    end

    run_analysis(file_name, server_dns, download)
  end
end

def save_analysis(analysis)
  analysis.save "analysis/#{analysis.name}.json"
  analysis.save_zip "analysis/#{analysis.name}.zip"
end


# ********************* Start List of Tasks ***********************************
desc 'create the analysis files with more output'
task :setup do
  analysis, _run_options = get_project
  save_analysis(analysis)
end

desc 'Create a cluster, but do not submit a job'
task :create_cluster do
  analysis, run_options = get_project
  save_analysis(analysis)
  create_cluster(run_options)
end

desc "run analysis with customized options"
task :run_custom, [:target, :project, :download, :batch_run_method] do |t, args|
  args.with_defaults(target: 'aws', project: nil, download: false, batch_run_method: 'batch_run' )
  analysis, run_options = get_project(args[:project])
  run_options[:batch_run_method] = args[:batch_run_method]
  save_analysis(analysis)
  if args[:target].downcase == 'aws'
    create_cluster(run_options)
  end
  run_analysis(analysis, run_options, args[:target], args[:download])
end

desc "terminate running aws instances"
task :terminate do
  Dir['*.json'].each do |json|
    # Read the JSON
    aws_options = {
        ami_lookup_version: 2,
    }
    aws = OpenStudio::Aws::Aws.new(aws_options)
    aws.load_instance_info_from_file(json)

    puts "AWS server instance found with IP: #{aws.os_aws.server.ip}"
    puts "AWS worker instances found with IPs: #{aws.os_aws.workers.map { |w| w.ip }.join(', ')}"

    if aws.os_aws.server.ip
      print "Do you want to terminate the AWS instances node [Y/N]? "
      if $stdin.gets.chomp.downcase == 'y'
        puts "Terminating ..."
        aws.terminate
        File.delete(json)
        puts "Make sure to verify that the instances have terminated via the AWS Console!".cyan
      end
    end
  end
end

desc 'delete all projects on site'
task :delete_all do
  if File.exist?('server_data.json')
    # parse the file and check if the instance appears to be up
    json = JSON.parse(File.read('server_data.json'), symbolize_names: true)
    server_dns = "http://#{json[:server][:dns]}"

    # Project data
    options = {hostname: server_dns}
    api = OpenStudio::Analysis::ServerApi.new(options)
    api.delete_all
  else
    puts "There doesn't appear to be a cluster running"
  end
end

desc 'delete all projects on site'
task :delete_all_vagrant do
  # parse the file and check if the instance appears to be up
  server_dns = 'http://localhost:8080'

  # Project data
  options = {hostname: server_dns}
  api = OpenStudio::Analysis::ServerApi.new(options)
  api.delete_all
end

task :default do
  system('rake -sT') # s for silent
end

desc 'make csv file of measures'
task :create_measure_csv do
  require 'CSV'
  require 'bcl'

  b = BCL::ComponentMethods.new
  new_csv_file = './measures/local_measures.csv'
  FileUtils.rm_f(new_csv_file) if File.exist?(new_csv_file)
  csv = CSV.open(new_csv_file, 'w')
  Dir.glob('./**/measure.json').each do |file|
    puts "Parsing Measure JSON for CSV #{file}"
    json = JSON.parse(File.read(file), symbolize_names: true)
    b.translate_measure_hash_to_csv(json).each { |r| csv << r }
  end

  csv.close
end

desc 'update measure.xml files'
task :update_measure_xmls do
  # Path to openstudio CLI
  openstudio_cli = ENV['OPENSTUDIO_CLI']
  if openstudio_cli && File.exist?(openstudio_cli)
    Dir['./measures/**/*.xml'].each do |measure|
      puts "#{measure} :: #{File.dirname(measure)}"
      puts `#{openstudio_cli} measure -u "#{File.dirname(measure)}"`
    end
  else
    puts "OPENSTUDIO_CLI env var is not set. Download and install OpenStudio CLI and set OPENSTUDIO_CLI to binary"
  end
end

# desc "terminate any running instances on AWS"
# task :terminate do
#   if @data[:location].upcase == 'AWS'
#     puts "    Destroying Worker Nodes in Group #{@data[:group_id]}".red
#     @aws.terminate_instances(@data[:workers].map { |w| w[:id] })
#   end
#
#   puts "Log into the AWS console and verify that the instances have terminated".bold.red
# end

desc 'update measures from BCL'
task :update_measures do
  require 'bcl'

  # Blow away existing measures if they exist
  # FileUtils.rm_rf("measures") if Dir.exist? 'measures'

  FileUtils.mkdir_p('measures')

  bcl = BCL::ComponentMethods.new
  bcl.parsed_measures_path = './measures'
  bcl.login # have to do this even if you don't set your username to get a session

  query = 'NREL%20PNNL%2BBCL%2BGroup'
  begin
    success = bcl.measure_metadata(query, nil, true)
  rescue => e
    puts "[ERROR] downloading new measure. #{e.message}"
  end

  # delete the test files
  Dir.glob("#{bcl.parsed_measures_path}/**/tests").each do |file|
    puts "Deleting test file #{file}"
    FileUtils.rm_rf(file)
  end
end

desc 'setup problem, start cluster, and run analysis (will submit another job if cluster is already running)'
task :run do
  task(:run_custom).invoke('aws')
end

desc 'run vagrant'
task :run_vagrant do
  task(:run_custom).invoke('vagrant')
end

desc 'run local (localhost:8080)'
task :run_local do
  task(:run_custom).invoke('local')
end

desc 'run NREL24a'
task :run_NREL24a do
  task(:run_custom).invoke('nrel24a')
end

desc 'run NREL24b'
task :run_NREL24b do
  task(:run_custom).invoke('nrel24b')
end

desc "run NREL24"
task :run_NREL24 do |t, args|
  task(:run_custom).invoke('nrel24')
end

desc "run LCNode1"
task :run_lcnode1 do |t, args|
  task(:run_custom).invoke('http://10.60.7.61:8080')
end

