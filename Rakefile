require 'pp'
require 'json'
require 'csv'
require 'iana-registry.rb'

task :rsync => [:rfcmod_rsync, :manual_rsync]

task :rfcmod_rsync do
  sh "rsync -av rsync.ietf.org::everything-ftp/yang/rfcmod ."
end

task :manual_rsync do
  sources = %w{
    ieee802-types
    ieee802-dot1q-types
    ieee802-dot1q-bridge
    iana-icmpv4-types
    iana-icmpv6-types
    iana-ipv6-ext-types
    iana-ospf-functional-cap-bits
    iana-tls-cipher-suite-algs
    iana-sip-option-tags
    iana-ssh-encryption-algs
    iana-ssh-key-exchange-algs
    iana-ssh-mac-algs
    iana-ssh-public-key-algs
  }.map do |modname|
    "rsync.ietf.org::everything-ftp/yang/catalogmod/#{modname}\*.yang"
  end
  sh "rsync", "-av", *sources, "manual"
end

task :build_sids do
  Dir.chdir("build")
  
  reg = IANA::Registry.load("yang-sid")

  rfcmod_files = Dir["../rfcmod/*.yang"]

  filename_parse = rfcmod_files.map do |name|
    if m = name.match(/\A(?<prefix>.*?)(?<modname>[^\/@]+)(?:@(?<rev>.*))?\.yang\z/)
      [m[:prefix], m[:modname], m[:rev]]
    else
      warn "** unrecognized file name #{name}"
      nil
    end          
  end
  # pp filename_parse

  mods = Hash.new{|h,k| h[k] = []}

  filename_parse.each do |prefix, modname, rev|
    filename1 = "#{modname}#{rev && "@" + rev}"
    yang_filename = "#{prefix}#{filename1}.yang"
    modfile = File.read(yang_filename)
    if m = modfile.match(/\A(?<sub>sub)?module (?<given_modname>[-A-Za-z0-9]+) {/)
      # p m[:sub]
      unless m[:sub]
        fail [modname, m[:given_modname]].inspect unless modname == m[:given_modname]
        mods[modname] << [prefix, rev]
      end
    end
  end

  # ignoring trouble YANG files:
  mods.delete("iana-ssh-HNAME-algs")
  mods.delete("iana-template")
  mods.delete("ietf-template")
  mods.delete("ietf-voucher-request")

  lastmod = mods.map {|k, v| [k, *v.last]}
  # pp lastmod
  warn "#{lastmod.count} modules to look at"

  total_sids = 0
  needed_sids = 0
  
  sid_state = []

  CSV.open("../stats.csv", 'wb') do |csv|
    csv << %w[modname nsids nalloc iana_ep iana_sz comments]

    lastmod.each do |modname, prefix, rev|
      filename1 = "#{modname}#{rev && "@" + rev}"
      yang_filename = "#{prefix}#{filename1}.yang"
      full_sid_filename = "#{filename1}.sid"
      sid_filename = "#{modname}.sid"
      # pp yang_filename, sid_filename
      unless File.exist?(sid_filename) && 
             File.ctime(sid_filename) > File.ctime(yang_filename)
        command = "time pyang -p ../rfcmod:../manual --sid-finalize --sid-generate-file=80000:10000 #{yang_filename}"
        sh command
        begin
          File.rename(full_sid_filename, sid_filename)
        rescue Errno::ENOENT
          warn "** can't rename #{full_sid_filename} to #{sid_filename}"
          expanded_sid_filename = Dir["#{full_sid_filename[0..-5]}@20*.sid"].first
          if expanded_sid_filename
            warn "** renaming #{expanded_sid_filename} instead"
            File.rename(expanded_sid_filename, sid_filename) 
          else
            fail "** Not creating a submodule sid for #{sid_filename}, see main module sid"
          end
        end
      end

      ep = nil; sz = nil; comments = []
      registry_path = "//xmlns:registry[@id='yang-sid-range']/xmlns:record[xmlns:module/text()='#{modname}']"
      REXML::XPath.each(reg.root, registry_path, IANA::Registry::NS) do |x|
        ep = Integer(x.elements['value'].text)
        sz = Integer(x.elements['entry_size'].text)
        # pp [modname, ep, sz]
      end


      sids = JSON.load_file(sid_filename)
      nsids = sids["ietf-sid-file:sid-file"]["item"].count
      nalloc = (((nsids + 2)/3*4)+49)/50*50 # maybe do a better rounding algorithm
      total_sids += nalloc
      needed_sids += nsids
      if sz && nalloc != sz
        comments << "alloc mismatch: #{sz} != #{nalloc}"
      end
      ret = [sid_filename, nsids, nalloc, ep, sz, comments.join(" ")]
      # p ret
      sid_state << ret
      csv << ret
    end

  end
  
  waste = (total_sids-needed_sids).fdiv(total_sids) * 100
  warn "#{total_sids} allocated/to be allocated ranges for #{needed_sids} SIDs (#{"%.3g" % waste} % waste); top users:"
  warn sid_state.sort_by {-_1[2]}[0..2].inspect

end

task :default => [:rsync, :build_sids]

task :clean do
  filenames = Dir["build/*.sid"]
  filenames.each do |fn|
    File.delete(fn)
  end
end

task :install do
  sh "bundle install"
end

