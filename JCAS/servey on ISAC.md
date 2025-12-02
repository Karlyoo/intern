## A Survey on Integrated Sensing, Communication, and Computation (ISCC)
### 1. Introduction
  - Background: The forthcoming 6G generation aims to go beyond traditional data services to usher in an era of "Intelligent of Everything"
  - Core Vision: This vision requires the seamless integration of three fundamental modules
    - Sensing: For information acquisition.
    - Communication: For information sharing.
    - Computation: For information processing and decision-making.
  - Problem Statement: Existing partial integration techniques (like ICC, ISC, ISAC) have made strides but suffer from interdependent performance, creating resource competition for time, energy, and bandwidth
  - Solution: The paper proposes ISCC (Integrated Sensing, Communication, and Computation), offering a systematic perspective to integrate these modules comprehensively.
    
### 2. Foundations: Partial Integration Technologies
A. Integrated Communication and Computation (ICC)
Refers to the joint design of communication and computation processes
- Mobile Edge Computing (MEC): Offloads tasks to edge servers to enhance performance (e.g., energy efficiency, latency) by balancing on-device loads
- Over-the-Air Computation (AirComp): Computes a function of signals (e.g., summation) directly over the air using waveform superposition, significantly enhancing communication efficiency for tasks like federated learning.
  
B. Integrated Sensing and Computation (ISC)
Refers to the joint design of sensing transceivers and processing algorithms.
- Wireless (Radar) Sensing: Uses waveforms (FMCW, OFDM) and signal processing (FFT) to detect target distance and velocity.
- Multi-modal Sensing: Fuses data from radar, cameras, etc., using machine learning for better performance compared to single-modality sensing.
- Mobile Crowdsensing: Leverages built-in sensors on ubiquitous mobile devices for large-scale data collection.
  
C. Integrated Sensing and Communication (ISAC)
Jointly designs sensing and communication modules to share hardware and resources.
- Integrated Wireless Sensing and Communication: Such as Dual-Functional Radar Communication (DFRC) systems that share spectrum and hardware.
- Integrated Multi-modal Sensing and Communication: Uses multi-modal data to assist Channel State Information (CSI) acquisition, or uses communication to assist data fusion.
  
### 3. Motivations, Benefits, and Challenges of ISCC
Motivations - Addressing Three Shortages of Existing Tech:
- Low Resource Utilization: Resources are often shared only between two modules, leading to inefficiency.
- Mismatch Between Goals: The goal of the overall task often conflicts with individual module goals (e.g., maximizing throughput vs. minimizing inference error).
- Ignoring Tight Coupling: Sensing, communication, and computation are highly coupled in complex tasks (competing for time/energy), which existing techniques ignore.

Benefits :
- Symbiosis:
  - Sensing provides prior knowledge to enhance communication
  - Communication acts as an information-sharing pipeline
  - Computation offers algorithms to improve sensing and communication.
- High Resource Utilization: Allows waveforms to coexist or be shared among the three functions.
- High Task Performance: Allocates resources under a unified task goal rather than separate objectives.

Challenges: 
- Multi-objective Optimization: Difficult to balance metrics with different units (e.g., throughput vs. MSE).
- Inference Management: Managing interference between coexisting waveforms.
- High Signal Design Complexity: Designing triple-functional signals is complex.
- Lack of Unified Design Criteria: Lack of standard criteria for task-oriented goals

### 4. Key Techniques: Signal Design
To enhance resource utilization, signal-level designs (beamforming and waveform) are investigated.

1.Single-functional Signal Design:
 - Independent signals dedicated to sensing, communication, and computation.
 - Drawback: Severe interference among signals.
2. Dual-functional Signal Design:
 - Designs one signal for sensing and another for AirComp.
 - Benefit: Reduces interference compared to single-functional designs
3. Triple-functional Signal Design (Air-ISCC):
 - Realizes sensing, communication, and computation in one signal.
 - Benefit: Eliminates interference among functions but increases design complexity.
   
### 5. Key Techniques: Network Resource Management
There are two main paradigms for resource management:

A. Joint RRM for Task Coexistence 
- Scenario: Sensing, communication, and computation tasks coexist with loose coupling.
- Method: Multi-objective optimization is formulated to guide resource allocation among the three to achieve multiple goals simultaneously.
  
B. Task-Oriented ISCC Resource Allocation 
- Scenario: For complex tasks where the three modules are tightly coupled.
- Use Cases:
  - Federated Learning: Convergence depends on sample count, sensing SNR, communication SNR, and computation speed; requires joint optimization.
  - Edge AI Inference: Accuracy depends on sensing quality, quantization error, and channel noise.
### 6. Future Applications
A. ISCC for Digital-Twins Enabled Wireless Networks
- Constructs virtual representations of physical systems.
- Hybrid AI Training: Integrates federated, split, and centralized learning to meet heterogeneous sensor requirements.
- Real-time Decision Making: Uses the digital twin for accurate network management

B. Computing Power Networks Supported ISCC 
- Micro-service Architecture: Decomposes signal processing into fine-grained blocks (e.g., filtering, feature extraction) shared across tasks.
- Directed-Graph Scheduling: Models nodes as a graph to find the best path for sensing, communication, and computation flow.

C. Space-Air-Ground Integrated Networks (SAGIN) 
- Terrestrial Sensing: UAVs collect ground data and forward pre-processed features to satellites for global fusion.
- Remote Sensing: Satellites perform multi-modal sensing directly and fuse data with ground stations

### 7. Unresolved Issues 
- Lack of Theoretical Analysis Tools: No unified theory to characterize the integrated performance of all three modules.
- Hardware Design: Requires hardware that breaks barriers between sensing and computation.
- Protocols and Standardization: Need protocols compatible with existing wireless systems.
- Security & Privacy: Integration increases risk of information leakage (e.g., inferring sensing data from comms load).
- Robustness: Systems must handle device failures or function loss.
