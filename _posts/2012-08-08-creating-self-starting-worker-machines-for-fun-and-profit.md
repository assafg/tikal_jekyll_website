---
layout: post
title: Creating self starting worker machines for fun and profit
created: 1344437586
author: avit
permalink: /ruby/creating-self-starting-worker-machines-fun-and-profit
tags:
- Ruby
---
<p>Lately, I needed to create worker machines (Machines that have Resque workers on them).</p>

<p>Usually, the process is very manual and tedious, you need to deploy the code to each machine (on code change), you need to start/stop/restart the workers and more.</p>

<p>Since I needed A LOT of workers, I really wanted to make the process automatic.</p>

<p>I wanted the machine to be in charge of everything it needs.</p>

<p>Meaning</p>

<ul>
<li>Pull latest git code</li>
<li>Copy file to the <code>current</code> directory</li>
<li>Startup god (which will start the Resque processes)</li>
</ul>


<p>I wanted the machine to be in charge of all this when it boots up, so in case I need to update the code or something, all I need is to reboot the machines and that’s it.</p>

<p>Also, scaling up the process is super easy, all I need is to add more machines.</p>

<p>The logic behind it was also being able to use spot-instances on Amazon, which are much cheaper.</p>

<p>So, as always, I thought I would document the process and open source it.</p>

<h2>Challenges</h2>

<p>What are the challenges we are looking at, when we want to do something like that.</p>

<ol>
<li>Awareness of RVM, Ruby version</li>
<li>Awareness of Bundler and Gem versions.</li>
</ol>


<p>Those are things we need to consider.</p>

<h2>Breaking down the process into pseudo code.</h2>

<p>So, let’s break down the process into pseudo code</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>Go to the temp folder
</span><span class='line'>git pull from origin + branch you want
</span><span class='line'>Copy all files from temp folder to current folder
</span><span class='line'>Go To current folder, bundle install
</span><span class='line'>Start God</span></code></pre></td></tr></table></div></figure>


<h2>RVM Wrappers</h2>

<p>For the last 2 parts of the process, if you try to do just <code>bundle install --deployment</code> for example, the script will fail, it does not know what bundle is, it resolves to the default Ruby installed on the machine at this point.</p>

<p>Since want the last 2 processes to go through RVM, we need to create a wrapper script.</p>

<h3>Creating RVM Wrappers</h3>

<p>You can create RVM wrappers like this:</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>rvm wrapper {ruby_version@gemset} {prefix} {executable}</span></code></pre></td></tr></table></div></figure>


<p>For example</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>rvm wrapper 1.9.2 bootup god</span></code></pre></td></tr></table></div></figure>


<p>This will generate this file: (called <code>bootup_god</code>)</p>

<figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
<span class='line-number'>6</span>
<span class='line-number'>7</span>
<span class='line-number'>8</span>
<span class='line-number'>9</span>
<span class='line-number'>10</span>
</pre></td><td class='code'><pre><code class='bash'><span class='line'><span class="c">#!/usr/bin/env bash</span>
</span><span class='line'>
</span><span class='line'><span class="k">if</span> <span class="o">[[</span> -s <span class="s2">"/usr/local/rvm/environments/ruby-1.9.2-p290"</span> <span class="o">]]</span>
</span><span class='line'><span class="k">then</span>
</span><span class='line'><span class="k">  </span><span class="nb">source</span> <span class="s2">"/usr/local/rvm/environments/ruby-1.9.2-p290"</span>
</span><span class='line'>  <span class="nb">exec </span>god <span class="s2">"$@"</span>
</span><span class='line'><span class="k">else</span>
</span><span class='line'><span class="k">  </span><span class="nb">echo</span> <span class="s2">"ERROR: Missing RVM environment file: '/usr/local/rvm/environments/ruby-1.9.2-p290'"</span> >&2
</span><span class='line'>  <span class="nb">exit </span>1
</span><span class='line'><span class="k">fi</span>
</span></code></pre></td></tr></table></div></figure>


<p>I did the same for bundle which generated this file: (called <code>bootup_bundle</code>)</p>

<figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
<span class='line-number'>6</span>
<span class='line-number'>7</span>
<span class='line-number'>8</span>
<span class='line-number'>9</span>
<span class='line-number'>10</span>
</pre></td><td class='code'><pre><code class='bash'><span class='line'><span class="c">#!/usr/bin/env bash</span>
</span><span class='line'>
</span><span class='line'><span class="k">if</span> <span class="o">[[</span> -s <span class="s2">"/usr/local/rvm/environments/ruby-1.9.2-p290"</span> <span class="o">]]</span>
</span><span class='line'><span class="k">then</span>
</span><span class='line'><span class="k">  </span><span class="nb">source</span> <span class="s2">"/usr/local/rvm/environments/ruby-1.9.2-p290"</span>
</span><span class='line'>  <span class="nb">exec </span>bundle <span class="s2">"$@"</span>
</span><span class='line'><span class="k">else</span>
</span><span class='line'><span class="k">  </span><span class="nb">echo</span> <span class="s2">"ERROR: Missing RVM environment file: '/usr/local/rvm/environments/ruby-1.9.2-p290'"</span> >&2
</span><span class='line'>  <span class="nb">exit </span>1
</span><span class='line'><span class="k">fi</span>
</span></code></pre></td></tr></table></div></figure>


<p>The files are created in <code>vim /usr/local/rvm/bin/</code>.</p>

<p>Now that we have the wrappers, we made sure the boot up process will be aware of RVM and we can continue.</p>

<h2>RC files</h2>

<p>If you want something to run when the machine starts you need to get to know the <code>rc</code> files.</p>

<p>Usually, those files are located in <code>/etc/</code> folder (may vary of course)</p>

<p>The files are loaded and executed in a specific order</p>

<figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
</pre></td><td class='code'><pre><code class='bash'><span class='line'>rc
</span><span class='line'>rc<span class="o">{</span>numer<span class="o">}</span>
</span><span class='line'>rc.local
</span></code></pre></td></tr></table></div></figure>


<p>If you want something to run when ALL the rest of the machine bootup process ran, you put it at the end of the rc.local file.</p>

<p>One gotcha here is that the file <em>must</em> end with <code>exit 0</code>, or your script will not run at all, and good luck finding out why :P</p>

<p>I put this line at the end, right before the <code>exit 0</code></p>

<figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
</pre></td><td class='code'><pre><code class='bash'><span class='line'>sh /root/start_workers.sh
</span></code></pre></td></tr></table></div></figure>


<h2>Finalize</h2>

<p>Now that I have the wrappers, I have the line in rc.local I can finalize the process with creating the shell script.</p>

<p>This is what the script looks for me</p>

<figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
<span class='line-number'>6</span>
<span class='line-number'>7</span>
<span class='line-number'>8</span>
</pre></td><td class='code'><pre><code class='bash'><span class='line'><span class="nb">cd</span> <span class="o">{</span>temp_folder<span class="o">}</span>
</span><span class='line'>git pull origin <span class="o">{</span>branch_name<span class="o">}</span>
</span><span class='line'><span class="nb">cd</span> ..
</span><span class='line'>cp -apRv <span class="o">{</span>source_folder<span class="o">}</span>/* <span class="o">{</span>destination_folder<span class="o">}</span> --reply<span class="o">=</span>yes
</span><span class='line'><span class="nb">echo</span> <span class="s2">"Running bundle install"</span>
</span><span class='line'><span class="nb">cd</span> /root/c <span class="o">&&</span> /usr/local/rvm/bin/bootup_bundle install --path /mnt/data-store/html/gogobot/shared/bundle --deployment --without development <span class="nb">test</span>
</span><span class='line'><span class="nb">echo</span> <span class="s2">"Starting RVM"</span>
</span><span class='line'>/usr/local/rvm/bin/bootup_god -c /root/c/god/resque_extra_workers.god
</span></code></pre></td></tr></table></div></figure>


<p>The god file to start the workers is standard.</p>

<figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
<span class='line-number'>6</span>
<span class='line-number'>7</span>
<span class='line-number'>8</span>
<span class='line-number'>9</span>
<span class='line-number'>10</span>
<span class='line-number'>11</span>
<span class='line-number'>12</span>
<span class='line-number'>13</span>
<span class='line-number'>14</span>
<span class='line-number'>15</span>
<span class='line-number'>16</span>
<span class='line-number'>17</span>
<span class='line-number'>18</span>
<span class='line-number'>19</span>
<span class='line-number'>20</span>
<span class='line-number'>21</span>
<span class='line-number'>22</span>
<span class='line-number'>23</span>
<span class='line-number'>24</span>
<span class='line-number'>25</span>
<span class='line-number'>26</span>
<span class='line-number'>27</span>
<span class='line-number'>28</span>
<span class='line-number'>29</span>
<span class='line-number'>30</span>
<span class='line-number'>31</span>
<span class='line-number'>32</span>
<span class='line-number'>33</span>
<span class='line-number'>34</span>
<span class='line-number'>35</span>
<span class='line-number'>36</span>
<span class='line-number'>37</span>
<span class='line-number'>38</span>
<span class='line-number'>39</span>
<span class='line-number'>40</span>
<span class='line-number'>41</span>
<span class='line-number'>42</span>
<span class='line-number'>43</span>
<span class='line-number'>44</span>
<span class='line-number'>45</span>
<span class='line-number'>46</span>
<span class='line-number'>47</span>
<span class='line-number'>48</span>
<span class='line-number'>49</span>
<span class='line-number'>50</span>
<span class='line-number'>51</span>
<span class='line-number'>52</span>
<span class='line-number'>53</span>
<span class='line-number'>54</span>
<span class='line-number'>55</span>
<span class='line-number'>56</span>
<span class='line-number'>57</span>
<span class='line-number'>58</span>
<span class='line-number'>59</span>
<span class='line-number'>60</span>
<span class='line-number'>61</span>
<span class='line-number'>62</span>
<span class='line-number'>63</span>
<span class='line-number'>64</span>
<span class='line-number'>65</span>
<span class='line-number'>66</span>
<span class='line-number'>67</span>
<span class='line-number'>68</span>
<span class='line-number'>69</span>
<span class='line-number'>70</span>
<span class='line-number'>71</span>
<span class='line-number'>72</span>
<span class='line-number'>73</span>
<span class='line-number'>74</span>
<span class='line-number'>75</span>
<span class='line-number'>76</span>
<span class='line-number'>77</span>
<span class='line-number'>78</span>
<span class='line-number'>79</span>
<span class='line-number'>80</span>
<span class='line-number'>81</span>
<span class='line-number'>82</span>
</pre></td><td class='code'><pre><code class='ruby'><span class='line'><span class="n">rails_env</span>   <span class="o">=</span> <span class="no">ENV</span><span class="o">[</span><span class="s1">'RAILS_ENV'</span><span class="o">]</span>  <span class="o">||</span> <span class="s2">"production"</span>
</span><span class='line'><span class="n">rails_root</span>  <span class="o">=</span> <span class="no">ENV</span><span class="o">[</span><span class="s1">'RAILS_ROOT'</span><span class="o">]</span> <span class="o">||</span> <span class="s2">"/mnt/data-store/html/gogobot/current"</span>
</span><span class='line'><span class="no">WORKER_TIMEOUT</span> <span class="o">=</span> <span class="mi">60</span> <span class="o">*</span> <span class="mi">10</span> <span class="c1"># 10 minutes</span>
</span><span class='line'>
</span><span class='line'><span class="c1"># Stale workers</span>
</span><span class='line'><span class="no">Thread</span><span class="o">.</span><span class="n">new</span> <span class="k">do</span>
</span><span class='line'>  <span class="kp">loop</span> <span class="k">do</span>
</span><span class='line'>    <span class="k">begin</span>
</span><span class='line'>      <span class="sb">`ps -e -o pid,command | grep [r]esque`</span><span class="o">.</span><span class="n">split</span><span class="p">(</span><span class="s2">"</span><span class="se">\n</span><span class="s2">"</span><span class="p">)</span><span class="o">.</span><span class="n">each</span> <span class="k">do</span> <span class="o">|</span><span class="n">line</span><span class="o">|</span>
</span><span class='line'>        <span class="n">parts</span>   <span class="o">=</span> <span class="n">line</span><span class="o">.</span><span class="n">split</span><span class="p">(</span><span class="s1">' '</span><span class="p">)</span>
</span><span class='line'>        <span class="k">next</span> <span class="k">if</span> <span class="n">parts</span><span class="o">[-</span><span class="mi">2</span><span class="o">]</span> <span class="o">!=</span> <span class="s2">"at"</span>
</span><span class='line'>        <span class="n">started</span> <span class="o">=</span> <span class="n">parts</span><span class="o">[-</span><span class="mi">1</span><span class="o">].</span><span class="n">to_i</span>
</span><span class='line'>        <span class="n">elapsed</span> <span class="o">=</span> <span class="no">Time</span><span class="o">.</span><span class="n">now</span> <span class="o">-</span> <span class="no">Time</span><span class="o">.</span><span class="n">at</span><span class="p">(</span><span class="n">started</span><span class="p">)</span>
</span><span class='line'>
</span><span class='line'>        <span class="k">if</span> <span class="n">elapsed</span> <span class="o">>=</span> <span class="no">WORKER_TIMEOUT</span>
</span><span class='line'>          <span class="o">::</span><span class="no">Process</span><span class="o">.</span><span class="n">kill</span><span class="p">(</span><span class="s1">'USR1'</span><span class="p">,</span> <span class="n">parts</span><span class="o">[</span><span class="mi">0</span><span class="o">].</span><span class="n">to_i</span><span class="p">)</span>
</span><span class='line'>        <span class="k">end</span>
</span><span class='line'>      <span class="k">end</span>
</span><span class='line'>    <span class="k">rescue</span>
</span><span class='line'>      <span class="c1"># don't die because of stupid exceptions</span>
</span><span class='line'>      <span class="kp">nil</span>
</span><span class='line'>    <span class="k">end</span>
</span><span class='line'>    <span class="nb">sleep</span> <span class="mi">30</span>
</span><span class='line'>  <span class="k">end</span>
</span><span class='line'><span class="k">end</span>
</span><span class='line'>
</span><span class='line'><span class="n">queue_name</span> <span class="o">=</span> <span class="s2">"graph_checkins_import"</span>
</span><span class='line'><span class="n">num_workers</span> <span class="o">=</span> <span class="mi">10</span>
</span><span class='line'>
</span><span class='line'><span class="c1"># FB wall posts</span>
</span><span class='line'><span class="n">num_workers</span><span class="o">.</span><span class="n">times</span> <span class="k">do</span> <span class="o">|</span><span class="n">num</span><span class="o">|</span>
</span><span class='line'>  <span class="no">God</span><span class="o">.</span><span class="n">watch</span> <span class="k">do</span> <span class="o">|</span><span class="n">w</span><span class="o">|</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">dir</span>      <span class="o">=</span> <span class="s2">"</span><span class="si">#{</span><span class="n">rails_root</span><span class="si">}</span><span class="s2">"</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">name</span>     <span class="o">=</span> <span class="s2">"resque-</span><span class="si">#{</span><span class="n">num</span><span class="si">}</span><span class="s2">-</span><span class="si">#{</span><span class="n">queue_name</span><span class="si">}</span><span class="s2">"</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">group</span>    <span class="o">=</span> <span class="s2">"resque"</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">interval</span> <span class="o">=</span> <span class="mi">2</span><span class="o">.</span><span class="n">minutes</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">env</span>      <span class="o">=</span> <span class="p">{</span><span class="s2">"QUEUE"</span><span class="o">=></span><span class="s2">"</span><span class="si">#{</span><span class="n">queue_name</span><span class="si">}</span><span class="s2">"</span><span class="p">,</span> <span class="s2">"RAILS_ENV"</span><span class="o">=></span><span class="n">rails_env</span><span class="p">,</span> <span class="s2">"PIDFILE"</span> <span class="o">=></span> <span class="s2">"</span><span class="si">#{</span><span class="n">rails_root</span><span class="si">}</span><span class="s2">/tmp/resque_</span><span class="si">#{</span><span class="n">queue_name</span><span class="si">}</span><span class="s2">_</span><span class="si">#{</span><span class="n">w</span><span class="si">}</span><span class="s2">.pid"</span><span class="p">}</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">pid_file</span> <span class="o">=</span> <span class="s2">"</span><span class="si">#{</span><span class="n">rails_root</span><span class="si">}</span><span class="s2">/tmp/resque_</span><span class="si">#{</span><span class="n">queue_name</span><span class="si">}</span><span class="s2">_</span><span class="si">#{</span><span class="n">w</span><span class="si">}</span><span class="s2">.pid"</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">start</span>    <span class="o">=</span> <span class="s2">"cd </span><span class="si">#{</span><span class="n">rails_root</span><span class="si">}</span><span class="s2">/ && bundle exec rake environment resque:work QUEUE=</span><span class="si">#{</span><span class="n">queue_name</span><span class="si">}</span><span class="s2"> RAILS_ENV=</span><span class="si">#{</span><span class="n">rails_env</span><span class="si">}</span><span class="s2">"</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">log</span>      <span class="o">=</span> <span class="s2">"</span><span class="si">#{</span><span class="n">rails_root</span><span class="si">}</span><span class="s2">/log/resque_god.log"</span>
</span><span class='line'>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">uid</span> <span class="o">=</span> <span class="s1">'root'</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">gid</span> <span class="o">=</span> <span class="s1">'root'</span>
</span><span class='line'>
</span><span class='line'>    <span class="c1"># restart if memory gets too high</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">transition</span><span class="p">(</span><span class="ss">:up</span><span class="p">,</span> <span class="ss">:restart</span><span class="p">)</span> <span class="k">do</span> <span class="o">|</span><span class="n">on</span><span class="o">|</span>
</span><span class='line'>      <span class="n">on</span><span class="o">.</span><span class="n">condition</span><span class="p">(</span><span class="ss">:memory_usage</span><span class="p">)</span> <span class="k">do</span> <span class="o">|</span><span class="n">c</span><span class="o">|</span>
</span><span class='line'>        <span class="n">c</span><span class="o">.</span><span class="n">above</span> <span class="o">=</span> <span class="mi">350</span><span class="o">.</span><span class="n">megabytes</span>
</span><span class='line'>        <span class="n">c</span><span class="o">.</span><span class="n">times</span> <span class="o">=</span> <span class="mi">2</span>
</span><span class='line'>      <span class="k">end</span>
</span><span class='line'>    <span class="k">end</span>
</span><span class='line'>
</span><span class='line'>    <span class="c1"># determine the state on startup</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">transition</span><span class="p">(</span><span class="ss">:init</span><span class="p">,</span> <span class="p">{</span> <span class="kp">true</span> <span class="o">=></span> <span class="ss">:up</span><span class="p">,</span> <span class="kp">false</span> <span class="o">=></span> <span class="ss">:start</span> <span class="p">})</span> <span class="k">do</span> <span class="o">|</span><span class="n">on</span><span class="o">|</span>
</span><span class='line'>      <span class="n">on</span><span class="o">.</span><span class="n">condition</span><span class="p">(</span><span class="ss">:process_running</span><span class="p">)</span> <span class="k">do</span> <span class="o">|</span><span class="n">c</span><span class="o">|</span>
</span><span class='line'>        <span class="n">c</span><span class="o">.</span><span class="n">running</span> <span class="o">=</span> <span class="kp">true</span>
</span><span class='line'>      <span class="k">end</span>
</span><span class='line'>    <span class="k">end</span>
</span><span class='line'>
</span><span class='line'>    <span class="c1"># determine when process has finished starting</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">transition</span><span class="p">(</span><span class="o">[</span><span class="ss">:start</span><span class="p">,</span> <span class="ss">:restart</span><span class="o">]</span><span class="p">,</span> <span class="ss">:up</span><span class="p">)</span> <span class="k">do</span> <span class="o">|</span><span class="n">on</span><span class="o">|</span>
</span><span class='line'>      <span class="n">on</span><span class="o">.</span><span class="n">condition</span><span class="p">(</span><span class="ss">:process_running</span><span class="p">)</span> <span class="k">do</span> <span class="o">|</span><span class="n">c</span><span class="o">|</span>
</span><span class='line'>        <span class="n">c</span><span class="o">.</span><span class="n">running</span> <span class="o">=</span> <span class="kp">true</span>
</span><span class='line'>        <span class="n">c</span><span class="o">.</span><span class="n">interval</span> <span class="o">=</span> <span class="mi">5</span><span class="o">.</span><span class="n">seconds</span>
</span><span class='line'>      <span class="k">end</span>
</span><span class='line'>
</span><span class='line'>      <span class="c1"># failsafe</span>
</span><span class='line'>      <span class="n">on</span><span class="o">.</span><span class="n">condition</span><span class="p">(</span><span class="ss">:tries</span><span class="p">)</span> <span class="k">do</span> <span class="o">|</span><span class="n">c</span><span class="o">|</span>
</span><span class='line'>        <span class="n">c</span><span class="o">.</span><span class="n">times</span> <span class="o">=</span> <span class="mi">5</span>
</span><span class='line'>        <span class="n">c</span><span class="o">.</span><span class="n">transition</span> <span class="o">=</span> <span class="ss">:start</span>
</span><span class='line'>        <span class="n">c</span><span class="o">.</span><span class="n">interval</span> <span class="o">=</span> <span class="mi">5</span><span class="o">.</span><span class="n">seconds</span>
</span><span class='line'>      <span class="k">end</span>
</span><span class='line'>    <span class="k">end</span>
</span><span class='line'>
</span><span class='line'>    <span class="c1"># start if process is not running</span>
</span><span class='line'>    <span class="n">w</span><span class="o">.</span><span class="n">transition</span><span class="p">(</span><span class="ss">:up</span><span class="p">,</span> <span class="ss">:start</span><span class="p">)</span> <span class="k">do</span> <span class="o">|</span><span class="n">on</span><span class="o">|</span>
</span><span class='line'>      <span class="n">on</span><span class="o">.</span><span class="n">condition</span><span class="p">(</span><span class="ss">:process_running</span><span class="p">)</span> <span class="k">do</span> <span class="o">|</span><span class="n">c</span><span class="o">|</span>
</span><span class='line'>        <span class="n">c</span><span class="o">.</span><span class="n">running</span> <span class="o">=</span> <span class="kp">false</span>
</span><span class='line'>      <span class="k">end</span>
</span><span class='line'>    <span class="k">end</span>
</span><span class='line'>  <span class="k">end</span>
</span><span class='line'><span class="k">end</span>
</span></code></pre></td></tr></table></div></figure>


<h2>Profit</h2>

<p>Now, it’s a completely automatic process, all you need is to run as many machines as you need, once the machine is booted up, it will deploy code to itself, copy files, install gems and run workers.</p>

<p>If you want to restart the process, add machines or anything else, you can do it with the click of a button in the AWS UI.</p>

<h2>Comments? Questions?</h2>

<p>Please feel free</p>
