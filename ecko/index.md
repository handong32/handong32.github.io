# ECKO: Energy and Carbon control for Kubernetes Orchestration

As cloud services become increasingly latency-sensitive and data center energy usage rises, there is an urgent need to address both operational and embodied carbon cost. However, data centers often overprovision resources, resulting in resource under-utilization. These inefficiencies not only waste energy but also accelerate hardware refresh cycles, exacerbating embodied emissions. In this work, we present PAX, a performance and energy aware Kubernetes scheduler that leverages machine learning techniques. Specifically, we present preliminary results from using Bayesian optimization to optimize microservices across a heterogeneous cluster. PAX improves application performance compared to modern schedulers and enables carbon-conscious scheduling by dynamically placing workloads on old and new servers based on performance sensitivity. The results illustrate an opportunity to reduce operational carbon while extending server lifetimes to mitigate embodied emissions. Our approach highlights the potential of ML-enhanced scheduling as a mechanism for improving both resource efficiency and sustainability in modern cloud infrastructures.

![](/ecko/ecko.png)

Source: https://dl.acm.org/doi/10.1145/3757892.3757902

Related Links: 

https://github.com/publiusys/ecko
https://github.com/publiusys/EEHPA
