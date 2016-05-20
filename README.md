## ECoWare evaluation

### What is ECoWare

![https://raw.githubusercontent.com/deib-polimi/ecoware-evaluation/master/deploy.png](https://raw.githubusercontent.com/deib-polimi/ecoware-evaluation/master/deploy.png)
**Figure 1:**  *ECoWare architecture*

ECoWare is a middleware for developing self-adaptive containerized applications based on the well-known MAPE control loop using the architecture showed in Figure 1.



### Evaluation

We evaluated our work by using two web applications: RUBiS and Pwitter. RUBiS is a well-known Internet application benchmark that simulates an on-line auction site. Our deployment of RUBiS uses three tiers: a load balancer, a scalable pool of JBoss application servers that run Java Servlets, and a MySQL database. [Pwitter](https://github.com/deib-polimi/pwitter) was developed by us. It is a simple Twitter-like social network that stores pweets, i.e., texts that are not limited in length, and that are indexed by the polarity of their sentiment. Pwitter is written in Python and is also a 3-tier application. It has a load balancer, a scalable pool of Gunicorn application servers, and a Redis database.

For our experiments we implemented two application-specific sensors; they capture the data required to calculate the average response time of a JBoss Application Server and of a Gunicorn Application server, respectively. We also implemented two application-specific Adaptation Hooks to enable platform adaptation; these hooks dynamically reconfigure JBoss and Gunicorn to better exploit the available resources. This is done following industry best practices. For example, best practices for Gunicorn suggest that the number of workers be `(2 x num_cores) + 1`.

The goal of our experiments was to maintain the average response time of the application servers below a certain threshold. After a profiling phase, we set the threshold (SLA) to 0.6 seconds for both applications. Indeed, both are able to sustain this value —under various kinds of workloads— using from 1 to 10 cores (i.e., the maximum amount of resources that we can afford for the experiments). The experiments were conducted to answer the following questions:


- **Question 1**: if we only used VMs, and no containers, would ECoWare perform better or worse than the current state of practice (i.e., AWS AutoScaling)?.
- **Question 2**: if we also took into account containers, would ECoWare perform better or worse than with VMs only?.

Performance is evaluated using the  `core x second` metric, i.e., we calculate how many cores are used during the experiments. The lower the value, the better the adaptation works. To simulate a varying workload we used JMeter.

####Question 1. 
We compared ECoWare against Amazon EC2's AutoScaling capabilities. We made this choice because it is, in practice, the most widely adopted scaling solution. For both applications we focused on adapting the business logic tiers. We created two AutoScaling groups on EC2: one per application. These groups were configured to scale from 1 to 10 t2.small VM instances (each instance had 1 CPU core and 2GB of memory). Both AutoScaling groups were configured to use Amazon Elastic Load Balancers. The databases were deployed on m4.xlarge instances (each instance had 4 CPU cores, 16GB of memory). With this setup, and with the workload we used, the databases were over-allocated; they never became the bottleneck during the experiments.

The EC2 AutoScaling groups were each given two policies. The first added 1 VM instance when the average CPU utilization, for the entire group, was over 90% for $1$ interval of 1 minute. The second removed 1 VM instance when the average CPU utilization, for the entire group, was below 40% for 1 interval of 1 minute. These thresholds were chosen following real-world best practices. We also decided to set the duration of the adaptation control to be as fast as possible; this way the AWS AutoScaling system would more quickly react to workload changes. Note, however, that Amazon's AutoScaling does not activate a scaling action if another action is executing. This means that, for example, when a new VM is added we must wait for it to finish turning on, for it to boot-up, and for it to be linked to the load balancer.

We measured that, on average, it took a VM 150 seconds to complete its boot-up process; this was evaluated from the launch of the VM creation command to when it had successfully booted and linked to the Elastic Load Balancer. We set the control interval to 180 seconds, we measured the response time and averaged it over 30 seconds, we set the planner to always read the most up-to-date measured response time before computing the next plan. We also defined the set point of the planner to be 0.5 seconds, that is about 10% less than the SLA. This way the planner would be able to range near the set point without violating the SLA. Finally, we stimulated the two applications with three different workloads: *low*, *medium* and *high*. All the experiments lasted 75 minutes; the number of users for each interval (in minutes) is shown in Table 1. 

| Experiment    | 0-15 | 15-30 | 30-45 | 45-60 | 60-75 |
|---------------|:-------------:|:--------------:|:--------------:|:--------------:|:--------------:|
|  Pwitter/Low  |       10      |       20       |       40       |       20       |       10       |
|  Pwitter/Med  |       15      |       35       |       60       |       35       |       15       |
|  Pwitter/High |       20      |       50       |       100      |       50       |       20       |
|   RUBiS/Low   |       50      |       200      |       400      |       200      |       400      |
|   RUBiS/Med   |      100      |       300      |       600      |       300      |       100      |
|   RUBiS/High  |      100      |       500      |      1000      |       500      |       100      |
| Pwitter/Float |       20      |       40       |       20       |       40       |       20       |
**Table 1:**  *Workloads: number of users per time interval (in minutes).*

We repeated the experiments five times; we shall now report on the average values we obtained on Figure 2. 


![https://raw.githubusercontent.com/deib-polimi/ecoware-evaluation/master/exp1.png](https://raw.githubusercontent.com/deib-polimi/ecoware-evaluation/master/exp1.png)
![https://raw.githubusercontent.com/deib-polimi/ecoware-evaluation/master/exp1.png](https://raw.githubusercontent.com/deib-polimi/ecoware-evaluation/master/exp2.png)
![https://raw.githubusercontent.com/deib-polimi/ecoware-evaluation/master/exp1.png](https://raw.githubusercontent.com/deib-polimi/ecoware-evaluation/master/exp3.png)

**Figure 2:**  *Obtained results. (CNT stands for Containers)*


Figures 2(a)-(c) show AWS AutoScaling's behavior with Pwitter stimulated with the three different workloads. This behavior is compared against the experiments illustrated in Figures 2(d)-(f), which used ECoWare's adaptation capabilities. The horizontal dotted line is the SLA and the thin line is the response time; they both refer to the left y-axis. The thicker line represents the allocated cores, and refers to the right y-axis. The metric `core x second` corresponds to the area under the thicker line (cores).


AWS AutoScaling clearly over-allocates resources. This occurs even for *low* workloads, and even if we use a very high threshold for CPU utilization. This is because both JBoss and Gunicorn tend to use nearly 100% of their allocated CPUs, even with moderate workloads. Furthermore, this value is also affected by how quickly a spike in the number of users is reached.

ECoWare, on the other hand, allocates less cores on average. The SLA is only violated with the *high* workload for around 5 minutes; this is understandable, and due to the fact that the approach is reactive. After a large spike ECoWare over-allocates resources for one control interval, and then slowly deallocates them and converges to a stable value. This can be seen in Figure 2(d) at minute 35, and in Figures 2(e)-(f) around minutes 20 and 35. Though the two example applications are completely different, we see similar results in Figures 2(g)-(l), which focus on RUBiS. In this case ECoWare never incurs in violations, while AWS continues to over-allocate resources in all the scenarios. Since AWS can only add a static number of VMs when reacting to an increment in CPU utilization, it is easy to find a workload (with high spikes) in which AWS is too slow to allocate resources, causing various SLA violations to occur; we did not find it useful to show such a case, given the lack of space. Table 2 shows the results of these experiments using the `core x second` metric. ECoWare outperforms AWS by 107% on average when managing Pwitter, and by 49% on average when managing RUBiS.

|  Experiment  |  AWS  | ECoWare |  Gain |
|:------------:|:-----:|:-------:|:-----:|
|  Pwitter/Low | 18810 |   7920  | 138% |
|  Pwitter/Med | 19965 |   9090  | 120% |
| Pwitter/High | 20970 |  12930  |  62% |
|   RUBiS/Low  | 15390 |   8970  |  72% |
|   RUBiS/Med  | 16320 |  10830  |  50% |
|  RUBiS/High  | 20265 |  16290  |  24% |
**Table 2:**  *AWS vs ECoWare without containers. Values are in CPU `core x second`.*



####Question 2. 
 With the second set of experiments we wanted to assess the benefit of using containers, instead of focusing solely on VMs. To do this we compared ECoWare, as used previously, against a new deployment that used an Amazon m4.2xlarge VM instance (8 cores and 32GB of memory). On this machine we installed the dockerized versions of the business logic tiers of the two applications. Our hypothesis was that the two applications had different workloads. By this we mean that there had to be moments in time in which one application would request *a lot* of resources, while the other would not. This would allow us to exploit the advantages of using containers. We also only used one VM in our experiments to make the benefits of using containers emerge more clearly.

The use of containers allows ECoWare to have a faster control rate, since creating, updating, and terminating containers can be done in a matter of milliseconds (around 4 to 6ms). We parametrized the planner with a control interval of 20 seconds, we measured the response time and averaged it over 10 seconds, we set the planner to always read the most up-to-date measured response time before computing the next plan. We then simulated two different workloads for Pwitter and RUBiS simultaneously. For RUBiS we used the *high* workload, for Pwitter we used the *float* workload (see Table 1).

Figure 2(m) shows that ECoWare without containers violated the SLA on Pwitter and allocated a peak of 5 cores at minute 50. Figure 2(o), on the other hand, shows that ECoWare with containers never violated the SLA, and remained quite close to the optimal resource allocation (1 core for 20 users, 2 cores for 40 users). Similar results emerged with RUBiS. Working with VMs only (Figure 2(l)) required us to allocate up to 8 cores, while the use of containers (Figure 2(n)) allowed us to allocate a maximum of 6 cores (the optimal allocation for 1000 users is 5 cores). This is due to the different parametrization of the planner, and to the different control intervals. Table 3 shows the results in detail. If we aggregate the two experiments, ECoWare with containers outperforms ECoWare with VMs by 46%.

|   Experiment  |  VMs  | Containers | Gain |
|:-------------:|:-----:|:----------:|:----:|
| Pwitter/Float |  8685 |    6580    | 32% |
|   RUBiS/High  | 16290 |    10210   | 60% |

**Table 3:**  *ECoWare with VMs vs ECoWare with containers. Values are in CPU `core x second`.*
