===== Link to all related PRs/Issues =====
  * https://github.com/verilog-to-routing/vtr-verilog-to-routing/issues?q=assignee%3ABill-hbrhbr

===== Background =====

  * Reading:
    * VTR 8 paper: the main CAD system we want to test / have designs go through. http://www.eecg.utoronto.ca/~kmurray/vtr/vtr8_trets.pdf
    * Tatum timing analyzer:  http://www.eecg.utoronto.ca/~kmurray/tatum/fpt2018_tatum.pdf
      * The timing analysis references in this paper are also likely good to check out, as the tatum paper assumes quite a bit of background. I can also point you at some tutorial references on timing analysis if you'd like -- let me know.
  * Chapters 1 - 4 of arch_and_cad.pdf (my book). Part of Chapter 2 (timing analysis) and Chapter 3 (placement) are the most relevant (both have evolved, but this gives the basic idea) {{:vaughn:arch_and_cad.pdf|}}

  * Docs & tutorials:
    * The VTR project is at https://github.com/verilog-to-routing/vtr-verilog-to-routing/
    * Documentation is here: https://docs.verilogtorouting.org/en/latest/vtr/
    * Go through the New Developer Tutorial at https://docs.verilogtorouting.org/en/latest/dev/tutorials/new_developer_tutorial/
    * Read the graphics section at https://docs.verilogtorouting.org/en/latest/vpr/graphics/
    * Experiment with the graphics, try them out.  Update any incorrect documentation and note anything you find counter-intuitive

  * VPR compile options
    * **Fast**: make CMAKE_PARAMS="-DVTR_ASSERT_LEVEL=2 -DVTR_ENABLE_PROFILING=OFF -DVTR_ENABLE_SANITIZE=off -DVTR_IPO_BUILD=off -DWITH_BLIFEXPLORER=off" vpr -j16
    * **Regular**: make CMAKE_PARAMS="-DVTR_ASSERT_LEVEL=3 -DVTR_ENABLE_SANITIZE=off -DVTR_IPO_BUILD=off -DWITH_BLIFEXPLORER=on" vpr -j16
    * **Profile**： make CMAKE_PARAMS="-DVTR_ENABLE_PROFILING=ON" vpr -j16 && make vpr -no-pie
    * **Sanitized**: make CMAKE_PARAMS="-DVTR_ASSERT_LEVEL=3 -DVTR_ENABLE_SANITIZE=on -DVTR_IPO_BUILD=off -DWITH_BLIFEXPLORER=on" vpr -j16
    * **Debug**: make CMAKE_PARAMS="-DVTR_ASSERT_LEVEL=4 -DVTR_ENABLE_SANITIZE=on -DVTR_IPO_BUILD=off -DWITH_BLIFEXPLORER=on" vpr -j16
    * **Compile**: make CMAKE_PARAMS="-DVTR_ASSERT_LEVEL=3 -DWITH_BLIFEXPLORER=on -DVTR_ENABLE_STRICT_COMPILE=on -DVTR_IPO_BUILD=off" -j16

===== Plan =====
  * Incremental timing analysis (k.murray@mail.utoronto.ca)
    - **(Done)** Review the pull request that adds incremental timing analysis to VTR.  Is it sufficiently commented?  Any code that is hard to understand?  Send Kevin your feedback via an online code review at https://github.com/verilog-to-routing/vtr-verilog-to-routing/pull/1296
    - **(Done)** Make a regtest that runs with full timing analysis and incremental timing analysis and compare the result files are identical (diff the route files and diff the timing report files; should be the same).  Make this run on a few architectures and circuits 
        - Strong regression suite: flagship arch + stereovision 3 circuit
          - Best this as additional sanity checking in the ''strong_timing_update_type'' test
        - Nightly: Titan arch + neuron, flagship arch + LU8 
        - Regression test creation guide is at https://docs.verilogtorouting.org/en/latest/dev/developing/#adding-tests
    - **(Done)** Profile vtr when running (1) the incremental timing analyzer Kevin Murray just created in vtr vs. (2) the traditional (whole netlist) timing analyzer
       * What are the differences?  Incremental analysis has some overhead compared to bulk when we're analyzing the whole design infrequently (which vtr currently does), but we'd like to minimize that overhead
    - **(Skipped)** Tatum is parallel when run in bulk mode (whole netlist) but not in the new incremental mode.  Add parallelism to the incremental version
       * Study the code
       * Tatum uses Intel thread building blocks, so read up on them
       * Create a proposal, and we'll discuss
       * Then code, measure, iterate ...
    - **(Skipped)** Generate results on the relative speed and parallelism of the bulk timing analyzer vs. the incremental version, as a function of the fraction of the netlist that has had its delays changed
    - **(Done)** Make the router use incremental timing analysis, and optimize the router by making more routines incremental
       * Currently using bulk only, but it could benefit from incremental analysis
    - **(Done)** Remove the pres_cost variable to reduce the size of a hot data structure.
       * Leverage the trade-off between storing the variable V.S. calculating the value on the fly
    - **(Ongoing)** Leverage the ability to rapidly timing analyze small changes to the placement to improve timing optimize late in the placement (mohamed.elgammal@mail.utoronto.ca)
       * **(Done)** Add a new interface that maps atom netlist pins raw setup slacks to those of the clb netlist pins.
       * **(Done)** Lots of things can be tried here; I think a new cost function term that is based on a (weighted?) sum of the negative or near-negative slacks would be good
       * **(Pending)** Plus moves that focus on moving blocks with timing-critical connections
    - **(Ongoing)** Refactor/modularize place.cpp in order to reduce its size.
    
===== Done =====

==Aug 25th==
  - Reduced place.cpp from ~3300 to ~2800 by moving routines to other files. Also made the data structures in place.cpp global variables (will modify it into something like vpr_context.h in another PR).
  - Added Doxygen documentation for everything I added/moved. More documentation to add but I want to get this PR in.
  - PR links:
    - With PlacerSetupSlacks interface + Refactored place.cpp + Doxygen comments: https://github.com/verilog-to-routing/vtr-verilog-to-routing/pull/1450
    - With only PlacerSetupSlacks interface: https://github.com/verilog-to-routing/vtr-verilog-to-routing/pull/1501
  - Reviewed Mahshad's and David's PRs.

==Aug 18th==
  - Did leetcode to prepare for Intel PEY interview this afternoon
  - Verified that the setup slacks are indeed strictly decreasing in the setup slack analysis stage.
  - Made the setup slack analysis during quench a VPR option ""--placement_quench_slack_analysis on"" and updated documentation.
  - Have some questions about overused IPIN blocks.

==Aug 14th==
  - Finished implementing setup slack analysis placement algorithm (timing update on every single swap, and revert if rejected).
  - VTR benchmark results: https://drive.google.com/file/d/1JRfVQWIme2yldLiOQBPWuD9R2-akIrl_/view?usp=sharing

==Aug 7th==
  - Meeting notes from yesterday
 {{:bill:aug06.png|}}
  - Currently trying to code reverting several moves made by try_swap(). If I find it too difficult, I'll simply code reverting one move.
  - Cost function discussion
     * Originally I cared about how much each slack value has changed since the last timing info update and give them respective weights.
       - The weight giving method is similar to how connection criticalities are calculated (except the Softmax method produces more drastic weight differences).
       - Since a lot of sign polarity changes may occur, the shifting of values becomes a necessity.
     * New idea: 
       - Only the critical path delay is of the most interest, and later maybe the moves proposed only affect the critical path connections
       - The most simple way: checking if the worst slack (from the modified set) has gotten better. But this method fails to account for other changes.
       - Another thought: get two vectors of the modified slack values(before and after), and then sort them. Compare these two vectors directly.

==Jul 31st==
  - Finished VPR logging PR with Xifan
  - Tried out setup slack updates on small Verilog circuits that David gave me. (How to generate timing constraint SDC file?)

==Jul 28th==
  - New option for printing overused nodes info: **--max_reported_overused_rr_nodes**
  - PR link: https://github.com/verilog-to-routing/vtr-verilog-to-routing/pull/1455/files
  - Incremental acc_cost (rejected) PR link: https://github.com/verilog-to-routing/vtr-verilog-to-routing/pull/1396

==Jul 24th==
  - Swept **quench_recompute_divider**: 0, 8, 64, 1024
     * Small titan benchmarks (neuron+stereovision): https://drive.google.com/file/d/1Oknf25iu5vKHC49GCiIDl_CN0hndHddx/view?usp=sharing
     * Large titan benchmarks: https://drive.google.com/file/d/1r4m2UNou79haz3VV-UPUMzmU3HH2Gfm8/view?usp=sharing
  - Created PlacerSetupSlacks interface for one-to-one mappings between CLB pins and timing analyzer setup slacks. Also updated slack/criticality routines in place.cpp
     * https://github.com/verilog-to-routing/vtr-verilog-to-routing/pull/1450

==Jul 21st==
  - Using the TimingInfo related interface to fetch the setup slack at each sink pin.
     * Play around the data I have right now (connection_slack and proposed_connection_slack)
     * Following Kevin's suggestions and read up on the timing tags and timing analyzer
  - Sweeped the inner_loop_recompute_divider parameter: 0(default), 2, 8, 64. A higher number indicates a more accurate timing analysis for swapping.
     * https://drive.google.com/file/d/1qdDxQ2VKvg6jZtomZHyW_JI8leEJw9Pu/view?usp=sharing

==Jul 17th==
  - Allow repeated counting of overused nodes but reduce acc_fac by half:
     * VTR benchmarks: https://drive.google.com/file/d/1njRb3Yaw6zu7c8HETpgk-dwhzJxMLU4H/view?usp=sharing
     * Titan benchmarks: https://drive.google.com/file/d/1VDPspnpKDC6ekBdMNkQ_zL4nRyQIv3Dn/view?usp=sharing
  - Presentation on speeding up pres_cost and acc_cost calculation/storage: https://docs.google.com/presentation/d/1AzYrAjfM4uH0J5yOkahq-rbk4a0wiD5oKldp4xlrQtw/edit?usp=sharing

==Jul 14th==
  - Finalize the QoR testings using the new methods.
  - Going through place.cpp to pinpoint various places where the timing info can be updated. I will come up with a proposal soon.

==Jul 7th==
  - Refactored pres_cost calculation.
  - Made acc_cost calculation more incremental. All the overused nodes should be produced by
     * Rerouted nets
     * Pins adjusted by reserve_locally_used_opins
  - Merged the calculation of overuse info into the acc_cost incremental update routine. This also corrects a small bug that results from my previous PR.
  - PR: https://github.com/verilog-to-routing/vtr-verilog-to-routing/pull/1396 
  - Titan QoR: https://drive.google.com/file/d/1sC6LKS2eucCxTKCxbaHe739Q0nKq-ONY/view?usp=sharing

==Jun 28th==
  - Fixed a bug (heap structure within the PlacerTimingCosts class)
  - OveruseInfo Qor: Ran VTR benchmarks (vtr_reg_qor_chain) and Titan benchmarks (titan_quick_qor)
     * Failed to gather info for directrf and sparcT1_chip2. Run separately or get test data online
     * VTR benchmarks: https://drive.google.com/file/d/1PVIgyMGYtgoIytFgNkA99aUwlZncWt6P/view
     * Titan benchmarks: https://drive.google.com/file/d/1cOlwYne3owPUApwVj3e4ihL9rGn8wpeo/view?usp=sharing
  - One thing I noticed: The Titan benchmarks always generates the same routing results, but the VTR benchmarks don't
     * Attempting to route at <number> channels (binary search bounds: [<leftBound>, <rightBound>])
  - Trying to trackdown all updates and usages of **pres_cost** and **pres_fac**. Trying to remove **pres_cost** field from routing storage data (calculate every instance on the fly using **pres_fac**).
     * Calculation of congestion cost: **get_rr_cong_cost**
     * VPR visuals: **draw_routing_costs**
     * The cost functions are different: the former is a product while the latter is a sum. Maybe a bug?
       * Product: base_cost * acc_cost * pres_cost
       * Sum: base_cost + acc_cost + pres_cost

==Jun 26th==
  - The placer structure: Outer_Loop_SA->Inner_Placement_Loop->try_swap
     * The outer and the inner loop all have incremental timing updates already
     * Trying to understand swapping, i.e. how are proposed moves' results evaluated? How are moves committed?
  - Will try out evaluating QoR after I make some experimental changes to the placer code.

==Jun 23rd==
  - The incremental sta functionality is already being utilized by the router
  - Optimized **calculate_overuse_info**
  - Trying to look for other things to optimize (updating the delays of each sink, and other route tree related stuff)
  - Timed various parts of the function
     * Router actually doesn't take up much of the whole VPR runtime (Placer does. And there are other significant bottlenecks such as route_lookahead.)
     * Testing how effective the **Incremental STA** actually is, as well as the effectiveness of the **calculate_overuse_info** optimization on very large titan benchmarks.
     * Incr_sta **Titan + Bitcoin**: https://docs.google.com/spreadsheets/d/14LEYEWPEl9di-MrMdxd_zNbP1bBx8AvadN7W7ej1OG0/edit?usp=sharing
     * Incr_sta **Titan + LU230**: https://docs.google.com/spreadsheets/d/1l4gHHioHwzQpUN29DUiX2md4GUSAIfVDbLaWLgIQX58/edit?usp=sharing
     * Overuse **Titan + Bitcoin**: https://docs.google.com/spreadsheets/d/1sjoWJZXYqvidrWe38qcf4iTnT1gF0JVCma8l2Me5e9A/edit?usp=sharing
     * Overuse **Titan + LU230**: https://docs.google.com/spreadsheets/d/1V711Av3bjr5D8dN0EqmOAET7KoAYH3S7OpDcK9Pb5oE/edit?usp=sharing

==Jun 16th==
  - VTR profile presentation: https://docs.google.com/presentation/d/1WPMBpf2hm-kVEoK7JJqm8Vb5ThbtctMju51brq4qu0E/edit?usp=sharing
  - Profile pngs: https://drive.google.com/drive/folders/1jhKOI_3uzMyYxozxJFUohqy-_k8iXl7u?usp=sharing

==Jun 12th==
  - Finalized regression test PR
  - Opened VTR profiling documentation PR
  - Profiled flagship arch with Stereovision3 and LU8 circuits: https://drive.google.com/drive/folders/1jhKOI_3uzMyYxozxJFUohqy-_k8iXl7u?usp=sharing
     * Largest overhead occurs at:
        - **recompute_criticalities** in place.cpp. Cause: used **update_td_costs** instead of **comp_td_costs**.
        - **try_timing_driven_route_tmpl** in route_timing.cpp. Cause: used **ConcreteSetupHoldTimingInfo::update**.
        - These are necessary overhead.
     * Other overhead:
        - **load_delay_annotations** in pb_type_graph_annotations.cpp. Cause: "horrible code"

==Jun 9th==
  - Setting up my new desktop over the weekend. Migrating existing work files from my laptop.
  - Got the regression tests fully working. However,
     * Shouldn't have added an extra option to VPR just for adding these regression tests. (Already removed)
     * Refactored the added code so that it is independent of the original script flow.
     * Will let Kevin do a final check afterward.
  - Read through the profiling guide, and checked that all steps are working. However, it seems like there are syntax errors within my data files so that they cannot be properly displayed. Currently looking into this issue.

==Jun 5th==
  - Added all the regression tests needed (1 strong + 2 nightly)
  - Created a new VPR option called --timing_report_prefix.
  - I was trying to figure out why my PR failed the vtr_reg_strong_pack_disable online (passed locally). Waiting for a fix (maybe I should open up a bug report?)
  - The SHA256 hashes (vtr_digest.cpp) in various files are different probably due to different file names. I will see if I can fix that tonight.
  - Moving onto the next stage of the plan next week (profiling the incremental STA vs full STA and maybe diff-ing more detailed files).

==Jun 2nd==
  - Got help from Kevin on a bunch of stuff (Thanks!)
  - Understood the differences between VTR running scripts: run_vtr_flow.pl, run_vtr_task.pl, parse_vtr_flow.pl, parse_vtr_task.pl
  - Understood what the golden results are (it's a copy of parse_results.txt, and I need to disable line wrapping so it becomes a nice table format)
  - Added a new option -check_incremental to **run_vtr_flow.pl** that can diff the files generated by full/incremental timing analysis. Runs are differentiated by the option --timing_update_type full/incremental.
     * ***.net:** Final packing file generated by the option **--net_file [file_name]**. SHA256 hash difference for **atom_netlist_id**.
     * ***.place:** Final placement file generated by the option **--place_file [file_name]**. SHA256 hash difference for **Netlist_ID**.
     * **report_timing.setup.post_place.rpt:** Intermediate timing file generated after placement and before routing. Identical.
     * ***.route:** Final routing file generated by the option **--route_file [file_name]**. Substantially different since the incr_sta is not integrated into the routing yet.
     * **Other post-route timing reports:** Also substantially different. I noticed that a lot of the timing statistics are the same, but the names/numberings of names, registers, LUTs are different.
  - Current:
     * Passed Flagship+Stereo3 and Flagship+LU8
     * If fail to pass, $error_code == 1 and $error_status = "failed: vpr full or incremental timing analysis do not produce the same results"
     * Opened a new pull request: https://github.com/verilog-to-routing/vtr-verilog-to-routing/pull/1329
  - Next steps:
     * Check how to access the neuron benchmark.

==May 29th==
  - Finished adding PuTTY setup tutorial to the computer infrastructure setup page.
  - Composed letter for GSoC opening and received revisions. I will send it to the mailing list later.
  - Played around with the VPR interface, including netlists and CLBs.
  - Cloned and compiled both the master branch and the incr_sta branch to run regression tests.
     * Found files for FLAGSHIP and TITAN archs, as well as blif/verilog files for LU8 and stereovision3.
     * Verified that the produced timing reports (setup/hold) and placer, router files are the same (except for filename hash).
     * Cannot find either blif or verilog file for **neuron** benchmark.
     * Will add all 3 regression tests to Strong and Nightly over the weekend (via a Pull Request?)

==May 26th==
  - Read through initial_placement.cpp, and dived deeper into physical_types.h to understand various arch and logic structs
  - I tried installing VTR on my Windows Cygwin, encountered a POSIX library bug when building. 
     * The bug report is here: https://github.com/verilog-to-routing/vtr-verilog-to-routing/issues/1317
     * Related Stack Overflow Post: https://stackoverflow.com/questions/33696092/whats-the-correct-replacement-for-posix-memalign-in-windows
     * Waiting for the Google developer to comment on it. If my Cygwin settings are all correct and there's no workaround, I'll try to fix the bug.
  - Set up shh key-pair authentication, multi-hop ssh, and VNC via PuTTY.
     * These two websites were helpful
         - https://blog.mario-mohr.net/2016/12/simple-multi-hop-ssh-connections-with-putty
         - https://the.earth.li/~sgtatham/putty/0.73/htmldoc/Chapter8.html#pubkey-puttygen
     * Took me long enough since it's not documented on the **Computer Infrastructure Setup** page (all Linux instructions). 
     * I can write a PuTTY setup tutorial for Windows users if it'll be helpful.
     * {{:bill:vnc.jpg|}}
  - Read through VTR's Document on Running Tests and successfully ran functionality tests (vtr_reg_basic && vtr_reg_strong)
  - Read through VTR Quick Start Document but encountered problem when following the VTR flow tutorial.
     * My PChenry/Wintermute account doesn't have GLIBC 2.27 installed, which is required for VPR.
     * I tried installing myself (https://ftp.gnu.org/gnu/glibc/), but the make install is failing with errors that didn't turn up in Google searches. 
     * It looks like I need something called linuxthreads, but they don't have the matching 2.27 distribution.
     * I will probably start from scratch again with someone's help.

==May 18th==
  - Read through netlist.h, atom_netlist.h to understand how the netlist is stored (blocks, pins, ports, nets)
  - Understand VTR data structures such as StrongId, NdMatrix, vector_map, ragged_matrix, etc.
  - Started studying how the initial placement is made (initial_placement.cpp)

==May 14th==
  - Skim through the VTR 8 paper
  - Read "Architecture and CAD for Deep-Submicron FPGAs" Chapters 1-4
  - Started studying the VPR code base (vpr_api.cpp, place.cpp, etc.)


  
