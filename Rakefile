task default: :toc

desc "Generate table of content"
task :toc do
  readme = File.read('README.md')
  header, content = readme.split(/^# Table of Contents.*?\n#/m)
  content = "#" + content
  toc = generate_table_of_contents(content)

  new_readme = [header, "# Table of Contents\n", toc, content].join("\n")

  File.open('README.md', 'w') { |f| f.puts new_readme }
  puts "README.md has a brand new Table of Contents!"
end

# From http://stackoverflow.com/questions/11948245/markdown-to-create-pages-and-table-of-contents
def generate_table_of_contents(markdown)
  table_of_contents = ""
  i_section = 0
  # to track markdown code sections, because e.g. ruby comments also start with #
  inside_code_section = false
  markdown.each_line do |line|
    inside_code_section = !inside_code_section if line.start_with?('```')

    forbidden_words = ['Table of contents', 'define', 'pragma']
    next if !line.start_with?('#') || inside_code_section || forbidden_words.any? { |w| line =~ /#{w}/ }

    title = line.gsub("#", "").strip
    href = title.gsub(/(^[!.?:\(\)]+|[!.?:\(\)]+$)/, '').gsub(/[!.,?:; \(\)-]+/, "-").downcase

    bullet = line.count("#") > 1 ? " *" : "#{i_section += 1}."
    table_of_contents << "  " * (line.count("#") - 1) + "#{bullet} [#{title}](\##{href})\n"
  end
  table_of_contents
end
