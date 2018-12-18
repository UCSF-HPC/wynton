# Cluster Specifications

## Software environment

All nodes on the cluster runs [CentOS] 7 which is updated on a regular basis.
The job scheduler is SGE 8.1.9 ([Son of Grid Engine]) which provides [queues]({{ '/scheduler/queues.html' | relative_url }}) for both communal and lab-priority tasks.


## Hardware

### Compute Nodes

<dl id="hosttable-summary" class="dl-horizontal">
  <dt>Compute nodes</dt><dd id="hosttable-summary-nodes">{{ site.specs.nodes }}</dd>
  <dt>Physical cores</dt><dd id="hosttable-summary-cores">{{ site.specs.physical_cores }}</dd>
  <dt>CPU</dt><dd id="hosttable-summary-cpu">{{ site.specs.cpu_range }}</dd>
  <dt>RAM</dt><dd id="hosttable-summary-ram">{{ site.specs.ram_range }}</dd>
  <dt>Swap</dt><dd id="hosttable-summary-ram">{{ site.specs.swap_range }}</dd>
  <dt>Local <code>/scratch</code></dt><dd id="hosttable-summary-scratch">{{ site.specs.local_scratch_size_range }}</dd>
  <dt>Local <code>/tmp</code></dt><dd id="hosttable-summary-tmp">{{ site.specs.local_tmp_size }}</dd>
</dl>

Most compute nodes have Intel processors, while others have AMD processes.  Each compute node has a local drive, which is either a hard disk drive (HDD), a solid state drive (SSD), or even a Non-Volatile Memory Express (NVMe) drive.
For additional details on the compute nodes, see the <a href="#details">Details</a> section below.

The compute nodes can only be utilized by [submitting jobs via the scheduler]({{ '/scheduler/submit-jobs.html' | relative_url }}) - it is _not_ possible to explicitly log in to compute nodes.


### Login Nodes

The [cluster can be accessed]({{ '/get-started/access-cluster.html' | relative_url }}) via SSH to one of two login nodes:

1. {{ site.login1.name }}: `{{ site.login1.hostname }}`
2. {{ site.login2.name }}: `{{ site.login2.hostname }}`


### Data Transfer Nodes

For transferring large data files, it is recommended to use the dedicate data transfer node:

1. {{ site.transfer.name }}: `{{ site.transfer.hostname }}`

which has a 10 Gbps connection - providing a file transfer speed of up to (theoretical) 1.25 GB/s = 4.5 TB/h.  As the login nodes, the transfer node can be accessed via SSH.

_Comment_: You can also transfer data via the login nodes, but since those only have 1 Gbps connections, you will see much lower transfer rates.


### Development Nodes

The cluster has development nodes for the purpose of validating scripts, prototyping pipelines, compiling software, and more.  Development nodes [can be accessed from the login nodes]({{ '/get-started/development-prototyping.html' | relative_url }}).

Node                        | # Physical Cores |       CPU |      RAM | Local `/scratch` |
----------------------------|-----------------:|----------:|---------:|-----------------:|
{{ site.devel.name }} |                8 |  2.66 GHz |   16 GiB |        0.125 TiB |

The development nodes have Intel Xeon CPU E5430 @ 2.66 GHz processors and local solid state drives (SSDs).


## Scratch Storage

The Wynton cluster provides two types of scratch storage:

* Local `/scratch/` - <span id="hosttable-summary-scratch2"></span> storage unique to each compute node (can only be accessed from the specific compute node).

* Global `/wynton/scratch/` - approx. 200 TiB storage ([BeeGFS](https://www.beegfs.io/content/)) accessible from everywhere.

There are no per-user quotas in these scratch spaces.  Files not added or modified during the last two weeks will be automatically deleted on a nightly basis.  Note, files with old timestamps that were "added" to the scratch place during this period will _not_ be deleted, which covers the use case where files with old timestamps are extracted from tar.gz file.  (Details: `tmpwatch --ctime --dirmtime --all --force` is used for the cleanup.)


## User and Lab Storage

Each user may use up to 200 GiB disk space in the home directory.  Research groups can add additional storage space by either mounting their existing storage or purchase new.
**Important**, please note that the Wynton HPC storage is _not_ backed up.  Users and labs are responsible to back up their own data outside of Wynton.


## Network

The compute nodes are connected using 10 Gbps Ethernet.
The cluster connects to NSF's [Pacific Research Platform] at a speed of 100 Gbps.


## Details

### All Compute Nodes

<script src="https://d3js.org/d3.v3.min.js"><!-- ~150 kB --></script>
<script src="https://cdn.datatables.net/1.10.16/js/jquery.dataTables.min.js"><!-- ~80 kB --></script>
<script src="https://cdn.datatables.net/1.10.16/js/dataTables.bootstrap.min.js"><!-- 2 kB --></script>

<table id="hosttable">
</table>

<script type="text/javascript" charset="utf-8">
d3.text("{{ '/assets/data/host_table.tsv' | relative_udatarl }}", "text/csv", function(host_table) {
  // extract date from header comments
  var timestamp = host_table.match(/^[#] Created on: [^\r\n]*[\r\n]+/mg, '')[0];
  timestamp = timestamp.replace(/^[#] Created on: /g, '');
  timestamp = timestamp.replace(/ [^ ]+/g, ''); // keep only the date
  timestamp = timestamp.trim();
  d3.select("#hosttable-timestamp").text(timestamp);

  // drop header comments
  host_table = host_table.replace(/^[#][^\r\n]*[\r\n]+/mg, '');
  host_table = d3.tsv.parse(host_table);

  d3.text("https://raw.githubusercontent.com/UCSF-HPC/wynton-slash2/master/status/qstat_nodes_in_state_au.tsv", "text/csv", function(host_status) {
    
    // drop header comments
    host_status = host_status.replace(/^[#][^\r\n]*[\r\n]+/mg, '');
    host_status = d3.tsv.parse(host_status);

    var table = d3.select("#hosttable");
    var thead, tbody, tfoot, tr, td, td_status;
    var value, value2;
    var cores = 0, coreMin = 1e9, coreMax = -1e9;
    var cpuMin = 1e9, cpuMax = -1e9;
    var ram = 0, ramMin = 1e9, ramMax = -1e9;
    var scratch = 0, scratchMin = 1e9, scratchMax = -1e9;
  
    /* For each row */
    var nodes = 0;
    host_table.forEach(function(row) {
      /* Ignore column on /tmp size, iff it exists */
      delete row["Local `/tmp`"];
    
      if (nodes == 0) {
        tr = table.append("thead").append("tr");
	tr.append("th").text("Status");
        for (key in row) {
          value = key.replace(/\`/g, "");
  	  tr.append("th").text(value);
        }
        tbody = table.append("tbody");
      }
      tr = tbody.append("tr");
      td_status = tr.append("td");
      for (key in row) {
        value = row[key];
        td = tr.append("td").text(value);
	if (key == "Node") {
	  value2 = host_status.filter(function(d) { return d.queuename == value });
	  if (value2.length > 0) {
            td_status.text("⚠");  // ⚠ ✖
//	  } else {
//            td_status.text("");
          }
        }
      }
  	
  	/* Cores */
  	value = parseInt(row["# Physical Cores"]);
  	cores += value;
  	if (value <= coreMin) coreMin = value;
  	if (value >= coreMax) coreMax = value;
  
  	/* CPU */
  	value = parseFloat(row["CPU"].match(/[\d.]+/));
  	if (value <= cpuMin) cpuMin = value;
  	if (value >= cpuMax) cpuMax = value;
  
  	/* RAM */
  	value = parseFloat(row["RAM"].match(/[\d.]+/));
  	ram += value;
  	if (value <= ramMin) ramMin = value;
  	if (value >= ramMax) ramMax = value;
  
  	/* Scratch */
  	value = parseFloat(row["Local `/scratch`"].match(/[\d.]+/));
  	scratch += value;
  	if (value <= scratchMin) scratchMin = value;
  	if (value >= scratchMax) scratchMax = value;
  
      nodes += 1;	
    });
  
    var addFooter = false;
    if (addFooter) tr = table.append("tfoot").append("tr");
    value = nodes + " nodes";
    if (addFooter) tr.append("td").text(value);
    d3.select("#hosttable-summary-nodes").text(value);
  
    value = cores + " cores (" + coreMin + "-" + coreMax + " cores/node, avg. " + (cores/nodes).toFixed(1) + " cores/node)";
    if (addFooter) tr.append("td").text(value);
    d3.select("#hosttable-summary-cores").text(value);
  
    value = cpuMin + "-" + cpuMax + " GHz";
    if (addFooter) tr.append("td").text(value);
    d3.select("#hosttable-summary-cpu").text(value);
  
    value = ramMin + "-" + ramMax + " GiB (avg. " + (ram/nodes).toFixed(1) + " GiB/node or " + (ram/cores).toFixed(1) + " GiB/core)";
    if (addFooter) tr.append("td").text(value);
    d3.select("#hosttable-summary-ram").text(value);
  
    value = scratchMin + "-" + scratchMax + " TiB";
    if (addFooter) tr.append("td").text(value);
    d3.select("#hosttable-summary-scratch2").text(value);
    value += " (avg. " + (scratch/nodes).toFixed(2) + " TiB/node or " + (scratch/cores).toFixed(3) + " TiB/core)";
    d3.select("#hosttable-summary-scratch").text(value);
  
    $(document).ready(function() {
      $('#hosttable').DataTable({
        "pageLength": 25,
	"order": [[ 1, "asc" ]]
  	});
    });
  });
});
</script>

Source: [host_table.tsv] produced on <span id="hosttable-timestamp"></span> using [wyntonquery].


<style>
table {
  margin-top: 2ex;
  margin-bottom: 2ex;
}
tfoot {
  border-top: 2px solid #000;
  font-weight: bold;
}
ttr:last-child { border-top: 2px solid #000; }
</style>

[CentOS]: https://www.centos.org/
[Son of Grid Engine]: https://arc.liv.ac.uk/trac/SGE
[Pacific Research Platform]: https://ucsdnews.ucsd.edu/pressrelease/nsf_gives_green_light_to_pacific_research_platform
[host_table.tsv]: {{ '/assets/data/host_table.tsv' | relative_url }}
[wyntonquery]: https://github.com/UCSF-HPC/wyntonquery