# Simulate CSI-RS feedback using MATLAB
```
% csi_simulation_type1.m
% MATLAB simulate 5G NR CSI feedback by Type I codebook
% Basis on  3GPP TS 38.214 Section 5.2.2.2.1
clear; clc; close all;

% 1. define simulation parameters
nFrames = 1;              % a 10 ms frame
subcarrierSpacing = 15e3; % subcarrier spacing (15 kHz)
nSubcarriers = 52 * 12;   % 52 PRB * 12 subcarrier/PRB
nSymbolsPerSlot = 14;     % 14 symbols per slot
nSlotsPerFrame = 10;      % 10 slots per frame
nTxAnts = 4;              % numser of transmitting antennas
nRxAnts = 4;              % numser of receiving antennas
snr_dB = 50;              % SNR (dB)

% channel parameters
delaySpread = 300e-9;   % delayed expansion (300 ns)
dopplerShift = 50;      % Doppler shift (50 Hz)
samplingRate = nSubcarriers * subcarrierSpacing; % Sampling rate

% CSI parameters
csiRsPorts = nTxAnts;   % number of CSI-RS ports
csiRsDensity = 1;       % CSI-RS density (1 RE/PRB)

% Type I codebook parameters
N1 = 2;                 % horizontal dimension
N2 = 2;                 % vertical dimension
O1 = 4;                 % horizontal oversampling
O2 = 4;                 % vertical oversampling
```

- Defines a MATLAB simulation for 5G NR CSI feedback using the Type I codebook (3GPP TS 38.214, Sec 5.2.2.2.1).
- Simulation parameters:
  - Frame structure (subcarriers, slots, symbols)
  - Antenna configuration (Tx/Rx)
  - Channel conditions (delay spread, Doppler, SNR)
  - CSI-RS and codebook parameters

```
% 2. Generate CSI-RS signal
% Generate resource grid
resourceGrid = zeros(nSubcarriers, nSymbolsPerSlot, nTxAnts);

% Define CSI-RS positions 
% (simplify positions to ensure sufficient reference signals for each subcarrier)
csiRsIndices = [];
csiRsSymbols = [2, 5, 10]; 
for sc = 1:nSubcarriers
    for sym_idx = 1:length(csiRsSymbols)
        csiRsIndices = [csiRsIndices; sc, csiRsSymbols(sym_idx)]; 
    end
end

% Fill CSI-RS signal (random QPSK symbols)
csiRsSignal = (randi([0 1], size(csiRsIndices, 1), nTxAnts)*2 - 1 + ...
               1j * (randi([0 1], size(csiRsIndices, 1), nTxAnts)*2 - 1)) / sqrt(2);

for idx = 1:size(csiRsIndices, 1)
    scIdx = csiRsIndices(idx, 1);
    symIdx = csiRsIndices(idx, 2);
    resourceGrid(scIdx, symIdx, :) = csiRsSignal(idx, :);
end
```

- Resource Grid & CSI-RS Generation
  - Creates a resource grid (3D matrix: subcarriers × symbols × Tx antennas).
  - Defines CSI-RS positions (pilot tones) across time and frequency.
  - Fills those positions with random QPSK pilot symbols.
  - Simulates CSI reference signals broadcast from base station antennas.
    
```
% 3. Analog Channels
% Time and Frequency Domain Channels
taps = 5;               % multipath number
delays = delaySpread * (0:taps-1) / (taps-1); 
gains = (randn(taps, nTxAnts, nRxAnts) + 1j * randn(taps, nTxAnts, nRxAnts)) / sqrt(2); % Rayleigh Gain

% Frequency domain channels (based on delay and Doppler)
H = zeros(nSubcarriers, nTxAnts, nRxAnts);
for f = 1:nSubcarriers
    freq = (f-1) * subcarrierSpacing;
    for t = 1:taps
        phaseShift = exp(-1j * 2 * pi * freq * delays(t));
        doppler = exp(1j * 2 * pi * dopplerShift * (rand-0.5)); 
        H(f, :, :) = H(f, :, :) + gains(t, :, :) * phaseShift * doppler;
    end
end

% Apply channel (operate directly in frequency domain)
rxGrid = zeros(nSubcarriers, nSymbolsPerSlot, nRxAnts);
for sym = 1:nSymbolsPerSlot
    for sc = 1:nSubcarriers
        % Extract the transmitted signal vector from resourceGrid (1 x nTxAnts)
        tx_vec = squeeze(resourceGrid(sc, sym, :)).'; 
        % Extract the corresponding channel matrix (nTxAnts x nRxAnts)
        H_slice = squeeze(H(sc, :, :));
        % Compute received signal y = x * H
        rx_vec = tx_vec * H_slice;
        % Store into the received grid
        rxGrid(sc, sym, :) = rx_vec;
    end
end
```

- Channel Model
  - Models a frequency-selective Rayleigh fading channel with:
  - Multiple taps (delay spread → multipath effects).
  - Doppler shift (mobility effect).
  - Computes frequency-domain channe l response H(f, tx, rx) by summing multipath components with delay + Doppler phase shifts.
  - Simulates how transmitted CSI-RS passes through this fading channel.
- Received Signal
  - For each subcarrier & symbol:
  - Takes the transmitted vector (from resource grid).
  - Multiplies with channel matrix H (Tx→Rx).
  - Stores received signal in rxGrid.
  - Effectively: y = x * H.
    
```
% 4. Add noise
snr = 10^(snr_dB / 10);
signalPower = mean(abs(rxGrid(rxGrid~=0)).^2);
noisePower = signalPower / snr;
noise = sqrt(noisePower/2) * (randn(size(rxGrid)) + 1j * randn(size(rxGrid)));
rxGridNoisy = rxGrid + noise;
```
```
% 5. Channel estimation
% LS channel estimation
H_est = zeros(nSubcarriers, nTxAnts, nRxAnts);
% Perform channel estimation for each subcarrier
for sc = 1:nSubcarriers
    % Find all CSI-RS indices on this subcarrier
    idx_list = find(csiRsIndices(:, 1) == sc);
    
    if ~isempty(idx_list)
        % Transmitted CSI-RS signals (nCSI x nTxAnts)
        X = csiRsSignal(idx_list, :);
        
        % Corresponding received signals (nCSI x nRxAnts)
        csi_sym_indices = csiRsIndices(idx_list, 2);
        Y = squeeze(rxGridNoisy(sc, csi_sym_indices, :));
        % If only one reference signal, Y may become a row vector → transpose
        if length(idx_list) == 1
            Y = Y.';
        end

        % LS estimation: H_est = pinv(X) * Y
        if size(X, 1) >= nTxAnts
            Xdagger = pinv(X);
            H_est_sc = Xdagger * Y; % nTxAnts x nRxAnts
            H_est(sc, :, :) = H_est_sc;
        else
            % If insufficient reference signals, pseudoinverse cannot be computed → simplified handling
            % warning('Insufficient CSI-RS samples for subcarrier %d', sc);
            H_est(sc, :, :) = zeros(nTxAnts, nRxAnts);
        end
    end
end
% For simulation purposes: fill the unestimated channels with true values (or interpolate) to avoid errors in later computation
H_est(all(H_est == 0, [2 3]), :, :) = H(all(H_est == 0, [2 3]), :, :);
```
- Channel Estimation
  - Performs Least Squares (LS) channel estimation:
     - Uses transmitted pilots X and received pilots Y.
     - Estimates H_est = pinv(X) * Y.
     - For subcarriers without enough pilots, sets channel estimate to zero or fills with ground truth (to avoid errors later).
     - Output: estimated channel matrix H_est(f, tx, rx).
    
```
% 6. Compute CSI report
% Average channel across frequency domain
H_mean = mean(H_est, 1); % Average along subcarrier dimension, result: 1 x nTxAnts x nRxAnts
H_mean_squeezed = squeeze(H_mean); % nTxAnts x nRxAnts

% Compute RI
[~, S, ~] = svd(H_mean_squeezed);
s_diag = diag(S);
% Determine rank based on signal energy vs noise power
ri = min(sum(s_diag.^2 > noisePower), 2); 
if ri == 0, ri = 1; end % At least report RI=1
disp(['RI: ' num2str(ri)]);

% Compute PMI using Type I Codebook
bestSinr = -Inf;
bestPmi_W = [];
best_i1 = 0;
best_i2 = 0;
% Iterate through codebook
for i1 = 0:(O1*N1*O2*N2-1) % Iterate all possible i1
    for i2 = 0:3
        W = type1_codebook(nTxAnts, ri, i1, i2, N1, N2, O1, O2);
        if ~isempty(W) && all(size(W) == [nTxAnts, ri]) % Check W dimensions
             % SINR calculation: (Signal) / (Interference + Noise)
             % Here, assume W is ideal, interference = 0, so simplified to SNR
            signal_power_eff = norm(H_mean_squeezed * W, 'fro')^2;
            sinr_eff = signal_power_eff / noisePower;
            
            if sinr_eff > bestSinr
                bestSinr = sinr_eff;
                bestPmi_W = W;
                best_i1 = i1;
                best_i2 = i2;
            end
        end
    end
end

pmi = bestPmi_W;
if isempty(pmi)
    warning('No valid PMI found, using identity matrix');
    pmi = eye(nTxAnts, ri);
end
fprintf('Best PMI found with i1=%d, i2=%d\n', best_i1, best_i2);
disp('PMI (W matrix, first few rows):');
disp(pmi(1:min(4, size(pmi, 1)), :));

% Compute CQI
% SINR of effective channel after precoding
sinr_val = bestSinr; 
cqi = round(log2(1 + sinr_val)); % Simplified mapping
cqi = min(max(cqi, 1), 15); % CQI range [1, 15]

disp(['CQI: ' num2str(cqi)]);

% 7. Visualization
% Plot channel matrix magnitude
figure;
imagesc(abs(squeeze(H_est(10, :, :)))); % Plot channel of the 10th subcarrier
title('Estimated Channel Magnitude (Subcarrier 10)');
xlabel('Receive Antennas');
ylabel('Transmit Antennas');
colorbar;

% Plot singular values from SVD
figure;
stem(10*log10(s_diag.^2 / noisePower));
title('SNR per Layer (from SVD)');
xlabel('Layer Index');
ylabel('SNR (dB)');
grid on;
```

- CSI Report Computation
  - Average channel across frequency → H_mean.
  - Rank Indicator (RI):
    - Compute SVD of H_mean.
    - Count number of strong singular values above noise → RI.
  - Precoding Matrix Indicator (PMI):
    - Iterates through all Type I codebook candidates (W).
    - Picks the precoder maximizing effective SINR.
    - Saves best (i1, i2) indices and W matrix.
  - Channel Quality Indicator (CQI):
    - Computes SINR of best PMI.
    - Maps SINR → CQI (simplified: log2(1+SINR), capped [1,15]).
- Visualization
  - Heatmap of estimated channel magnitude for one subcarrier.
  - Stem plot of per-layer SNR (from SVD singular values).

## type 1 codebook function
```
function [W] = type1_codebook(nTxAnts, ri, i1, i2, N1, N2, O1, O2)
    isDualPolarized = (nTxAnts == 2*N1*N2);
    nPortsBase = N1 * N2; 
    
    if nTxAnts ~= N1*N2 && ~isDualPolarized
        W = [];
        return;
    end
    
    L = 2; % Each codeword is formed by combining two DFT beams
    
    % i1 determines the selection of the two DFT beams
    i1_1 = mod(floor(i1 / (O2 * N2)), O1 * N1); 
    i1_2 = mod(i1, O2 * N2);                    
    
    % *** Key fix: use ' to transpose into a column vector ***
    u1_1 = exp(-1j * 2 * pi * (0:N1-1)' * i1_1 / (O1 * N1)) / sqrt(N1);
    u2_1 = exp(-1j * 2 * pi * (0:N2-1)' * i1_2 / (O2 * N2)) / sqrt(N2);
    W1_base_1 = kron(u2_1, u1_1); % N1*N2 x 1 vector
    
    % Second beam (simplified as adjacent index)
    i1_1_2 = mod(i1_1 + 1, O1 * N1);
    u1_2 = exp(-1j * 2 * pi * (0:N1-1)' * i1_1_2 / (O1 * N1)) / sqrt(N1);
    u2_2 = u2_1;
    W1_base_2 = kron(u2_2, u1_2);
    
    % W1: beam selection matrix (nTxAnts x L)
    W1_base = [W1_base_1, W1_base_2]; % nPortsBase x L (e.g., 4x2)
    
    if isDualPolarized
        W1 = kron(eye(2), W1_base); % Simplified dual-polarized structure
    else
        W1 = W1_base;
    end
    
    % W2: beam combination coefficients (L x ri)
    if ri == 1
        phi = exp(1j * pi * i2 / 2); % Phase correction to match standard
        W2 = [1; phi] / sqrt(2); % 2x1 matrix
    elseif ri == 2
        if i2 == 0
            W2 = [1 0; 0 1]/sqrt(2);
        elseif i2 == 1
            W2 = [1 1; 1 -1]/sqrt(4);
        elseif i2 == 2
            W2 = [1 1; 1j -1j]/sqrt(4);
        else % i2 == 3
            W2 = []; % ri=2, i2=3 in single-panel is reserved
        end
    else
        W2 = [];
    end

    if isempty(W2)
        W = [];
        return;
    end
    
    % Final precoding matrix
    W = W1 * W2;
end
```

- nTxAnts: Total number of transmit antennas
- ri: Rank Indicator (number of transmission layers, 1 or 2)
- i1: First-layer index, determines beam direction
- i2: Second-layer index, determines beam combination method
- N1: Number of antennas in the horizontal direction (for single polarization)
- N2: Number of antennas in the vertical direction (for single polarization)
- O1: Oversampling factor in the horizontal direction
- O2: Oversampling factor in the vertical direction
```
i1 → [DFT horizontal index, DFT vertical index] → generate W1_base_1
                 | 
                 |  (horizontal index+1)
                 ↓
           generate W1_base_2
                 
W1_base_1 & W1_base_2 → W1 (depending on single/dual polarization)
RI, i2 → select W2 (combination coefficients)
W1 × W2 → Final precoding matrix W
```

### Output
<img width="426" height="219" alt="image" src="https://github.com/user-attachments/assets/d4390799-f76e-461e-9bef-ebe2bcc1a2a8" />
<img width="1416" height="626" alt="image" src="https://github.com/user-attachments/assets/f0825dea-8d53-4727-8c82-9985bb2454eb" />

- Figure 1 (Right): Estimated Channel Magnitude (Subcarrier 10)
  - Shows the estimated MIMO channel response at subcarrier 10.
  - Each color block represents the magnitude of the estimated channel coefficient ∣𝐻𝑡𝑥,𝑟𝑥∣.
  - Brighter colors (yellow) = stronger channel gain, darker colors (blue) = weaker gain.
    
This essentially visualizes the channel matrix for one subcarrier, to see which Tx–Rx antenna pairs have stronger paths.

- Figure 2 (Left): SNR per Layer (from SVD)
  - Shows the Signal-to-Noise Ratio (SNR) of each spatial layer, derived from Singular Value Decomposition (SVD) of the averaged channel.
  - Each point corresponds to the effective SNR of one singular mode of the MIMO channel.
  - Interpretation:
     - Layer 1 ≈ 27 dB (strongest spatial stream)
     - Layer 2 ≈ 24 dB
     - Layer 3 ≈ 18 dB
     - Layer 4 ≈ 5 dB (very weak, may not be used)

In practice, the Rank Indicator (RI) would be chosen based on how many layers have acceptable SNR, e.g., RI=2 or 3 here (since layer 4 is too weak).
