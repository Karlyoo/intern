# LDPC Simulation 
### 1. Parameter Settings
```
clear;
clc;

%% === Parameter Settings ===
SNR_dB_range = -3:1:2;   % SNR range (dB)
num_blocks   = 300;      % Number of simulation blocks
max_iter     = 75;       % Maximum BP iterations
n            = 64;       % Codeword length
k            = 32;       % Message length
d            = 0.3;      % Sparsity
m            = n - k;    % Number of parity bits
```

- **Purpose**: Initializes simulation parameters.
  - `clear; clc;`: Clears the workspace and command window for a clean simulation environment.
  - `SNR_dB_range`: Defines the SNR range in dB (-3 to 2 dB, step size 1 dB) for BER evaluation.
  - `num_blocks`: Specifies the number of codeword blocks (300) to simulate for statistical reliability.
  - `max_iter`: Sets the maximum number of Belief Propagation (BP) decoding iterations (75).
  - `n`: Codeword length (64 bits), representing the total length of the encoded bits.
  - `k`: Message length (32 bits), representing the information bits before encoding.
  - `d`: Sparsity (0.3), controlling the density of 1s in the parity-check matrix.
  - `m`: Number of parity bits (`n - k = 32`), corresponding to the number of check nodes in the LDPC code.
---

### 2. Construct Random Parity Matrix P
```
%% === Construct random parity matrix P ===
P = sparse(m, k);
for i = 1:m
    num_ones = max(1, round(d * k));
    cols = randperm(k, num_ones);
    P(i, cols) = 1;
end

% Ensure each variable node is connected to at least one check node
for j = 1:k
    if sum(P(:, j)) == 0
        row = randi(m);
        P(row, j) = 1;
    end
end
```
- **Purpose**: Generates a random sparse parity-check matrix `P` for the LDPC code.
  - `P = sparse(m, k)`: Initializes an `m × k` (32 × 32) sparse matrix for efficiency.
  - **Loop over rows** (`i = 1:m`):
    - `num_ones = max(1, round(d * k))`: Calculates the number of 1s per row based on sparsity `d` (0.3 × 32 ≈ 10 ones per row, ensuring at least one 1).
    - `cols = randperm(k, num_ones)`: Randomly selects `num_ones` column indices from 1 to `k` (32).
    - `P(i, cols) = 1`: Sets the selected columns in row `i` to 1, creating a sparse structure.
  - **Variable node check**:
    - Ensures each column (variable node) in `P` has at least one 1 to avoid unconnected variable nodes.
    - For each column `j`, if `sum(P(:, j)) == 0`, randomly selects a row (`row = randi(m)`) and sets `P(row, j) = 1`.

---

### 3. Construct H and G Matrices
```
%% === Construct H and G matrices ===
H = [P, speye(m)];          % Parity-check matrix
G = [speye(k), mod(P', 2)]; % Generator matrix
```

- **Purpose**: Constructs the parity-check matrix `H` and generator matrix `G` for LDPC encoding and decoding.
  - `H = [P, speye(m)]`:
    - Creates the parity-check matrix `H` (size `m × n`, 32 × 64) by concatenating:
      - `P`: The random sparse matrix (`m × k`, 32 × 32).
      - `speye(m)`: An `m × m` (32 × 32) identity matrix, representing parity bits.
    - `H` defines the LDPC code structure, where `H * codeword^T = 0` (mod 2) for valid codewords.
  - `G = [speye(k), mod(P', 2)]`:
    - Creates the generator matrix `G` (size `k × n`, 32 × 64) in systematic form:
      - `speye(k)`: A `k × k` (32 × 32) identity matrix for systematic bits.
      - `mod(P', 2)`: The transpose of `P` (modulo 2) for parity bits.
    - Used to encode messages: `codeword = msg * G (mod 2)`.

---

### 4. Main Simulation Loop
```
%% === Main simulation loop ===
BER = zeros(size(SNR_dB_range));

for idx = 1:length(SNR_dB_range)
    
    SNR_dB   = SNR_dB_range(idx);
    snr_lin  = 10^(SNR_dB / 10);
    noise_var = 1 / (2 * snr_lin);
    
    total_errors = 0;
    total_bits   = 0;
```

- **Purpose**: Iterates over the specified SNR range to compute BER for each SNR value.
  - `BER = zeros(size(SNR_dB_range))`: Initializes a BER array to store results for each SNR value.
  - **Loop over SNR values** (`idx = 1:length(SNR_dB_range)`):
    - `SNR_dB = SNR_dB_range(idx)`: Selects the current SNR value in dB.
    - `snr_lin = 10^(SNR_dB / 10)`: Converts SNR from dB to linear scale.
    - `noise_var = 1 / (2 * snr_lin)`: Calculates noise variance for the AWGN channel, assuming a signal power of 1 and QPSK modulation (2 bits per symbol).
    - `total_errors` and `total_bits`: Initialize counters for bit errors and total bits processed across blocks.

---

### 5. Random Message Generation and Encoding
```
    for blk = 1:num_blocks
        %% === Random message and encoding ===
        msg = randi([0 1], 1, k);
        codeword = mod(msg * G, 2);
```

- **Purpose**: Generates a random message and encodes it using the LDPC generator matrix.
  - `msg = randi([0 1], 1, k)`: Creates a random binary message of length `k` (32 bits).
  - `codeword = mod(msg * G, 2)`: Encodes the message into a codeword of length `n` (64 bits) by multiplying with `G` (modulo 2 for binary arithmetic).

---

### 6. QPSK Modulation
```
        %% === QPSK Modulation ===
        b0 = codeword(1:2:end);
        b1 = codeword(2:2:end);
        symbols = (1 - 2*b0) + 1j * (1 - 2*b1);
```

- **Purpose**: Modulates the codeword using QPSK (Quadrature Phase-Shift Keying).
  - `b0 = codeword(1:2:end)`: Extracts even-indexed bits (1st, 3rd, ...) for the real part.
  - `b1 = codeword(2:2:end)`: Extracts odd-indexed bits (2nd, 4th, ...) for the imaginary part.
  - `symbols = (1 - 2*b0) + 1j * (1 - 2*b1)`:
    - Maps bits to QPSK symbols: `0 → +1`, `1 → -1` for both real and imaginary parts.
    - Produces complex symbols (e.g., {1+1j, 1-1j, -1+1j, -1-1j}) for the 32-bit pairs (64 bits total → 32 symbols).

---

### 7. AWGN Channel
```
        %% === AWGN Channel ===
        noise = sqrt(noise_var) * (randn(size(symbols)) + 1j*randn(size(symbols)));
        rx = symbols + noise;
```

- **Purpose**: Simulates transmission over an AWGN channel by adding Gaussian noise.
  - `noise = sqrt(noise_var) * (randn(size(symbols)) + 1j*randn(size(symbols)))`:
    - Generates complex Gaussian noise with zero mean and variance `noise_var`.
    - `randn(size(symbols))`: Real and imaginary parts are independently generated with standard normal distribution, scaled by `sqrt(noise_var)`.
  - `rx = symbols + noise`: Adds noise to the transmitted QPSK symbols to simulate channel effects.

---

### 8. LLR Calculation
```
        %% === LLR Calculation ===
        LLR = zeros(n, 1);
        LLR(1:2:end) = 2 * real(rx) / noise_var;
        LLR(2:2:end) = 2 * imag(rx) / noise_var;
```

- **Purpose**: Computes Log-Likelihood Ratios (LLRs) for BP decoding.
  - `LLR = zeros(n, 1)`: Initializes an array for LLRs of all `n` (64) bits.
  - `LLR(1:2:end) = 2 * real(rx) / noise_var`: Computes LLRs for even-indexed bits (real part of QPSK symbols).
  - `LLR(2:2:end) = 2 * imag(rx) / noise_var`: Computes LLRs for odd-indexed bits (imaginary part).
  - **Formula**: For QPSK, LLR for a bit is proportional to the received symbol’s real/imaginary component divided by noise variance, assuming unit signal power.

---

### 9. Build Tanner Graph Adjacency Lists
```
        %% === Build Tanner graph adjacency lists ===
        [m, n] = size(H);
        N = cell(n, 1); % Variable nodes connected to which check nodes
        M = cell(m, 1); % Check nodes connected to which variable nodes
        
        for i = 1:m
            M{i} = find(H(i, :));
        end
        for j = 1:n
            N{j} = find(H(:, j))';
        end
```

- **Purpose**: Constructs adjacency lists for the Tanner graph representation of the LDPC code.
  - `[m, n] = size(H)`: Retrieves dimensions of `H` (32 × 64).
  - `N = cell(n, 1)`: Creates a cell array where `N{j}` lists check nodes connected to variable node `j`.
  - `M = cell(m, 1)`: Creates a cell array where `M{i}` lists variable nodes connected to check node `i`.
  - **Check node connections**:
    - `M{i} = find(H(i, :))`: Finds indices of non-zero entries in row `i` of `H` (variable nodes connected to check node `i`).
  - **Variable node connections**:
    - `N{j} = find(H(:, j))'`: Finds indices of non-zero entries in column `j` of `H` (check nodes connected to variable node `j`).

---

### 10. Initialize Messages for BP Decoding
```
        %% === Initialize messages ===
        msg_v2c = zeros(m, n); % Variable -> Check messages
        msg_c2v = zeros(m, n); % Check -> Variable messages
        est_codeword = zeros(1, n);
```

- **Purpose**: Initializes message arrays for Belief Propagation (BP) decoding.
  - `msg_v2c = zeros(m, n)`: Initializes variable-to-check messages (size 32 × 64) to zero.
  - `msg_c2v = zeros(m, n)`: Initializes check-to-variable messages (size 32 × 64) to zero.
  - `est_codeword = zeros(1, n)`: Initializes the estimated codeword (length 64) for decoding output.

---

### 11. BP Decoding Iterations
```
        %% === BP Decoding Iterations ===
        for iter = 1:max_iter
            % Variable to Check update
            for v = 1:n
                for c_idx = 1:length(N{v})
                    c = N{v}(c_idx);
                    others = N{v}(N{v} ~= c);
                    msg_v2c(c, v) = LLR(v) + sum(msg_c2v(others, v));
                end
            end
            
            % Check to Variable update
            for c = 1:m
                for v_idx = 1:length(M{c})
                    v = M{c}(v_idx);
                    others = M{c}(M{c} ~= v);
                    tanh_vals = tanh(msg_v2c(c, others)/2);
                    prod_tanh = prod(tanh_vals);
                    % Avoid overflow
                    prod_tanh = max(min(prod_tanh, 0.999999), -0.999999);
                    msg_c2v(c, v) = 2 * atanh(prod_tanh);
                end
            end
            
            % Combine total LLR and estimate codeword
            total_LLR = zeros(1, n);
            for v = 1:n
                total_LLR(v) = LLR(v) + sum(msg_c2v(N{v}, v));
            end
            est_codeword = double(total_LLR < 0);
            
            % Check syndrome
            syn = mod(H * est_codeword', 2);
            if all(syn == 0)
                break;
            end
        end
```
- **Purpose**: Implements Belief Propagation (BP) decoding using the sum-product algorithm.
  - **Loop over iterations** (`iter = 1:max_iter`):
    - Performs up to `max_iter` (75) iterations of message passing unless early termination occurs.
  - **Variable-to-Check Update**:
    - For each variable node `v`, updates messages to connected check nodes `c` in `N{v}`.
    - `msg_v2c(c, v) = LLR(v) + sum(msg_c2v(others, v))`: Computes outgoing message as the sum of the intrinsic LLR and incoming check-to-variable messages from other check nodes.
  - **Check-to-Variable Update**:
    - For each check node `c`, updates messages to connected variable nodes `v` in `M{c}`.
    - `tanh_vals = tanh(msg_v2c(c, others)/2)`: Applies hyperbolic tangent to incoming messages (divided by 2 for sum-product algorithm).
    - `prod_tanh = prod(tanh_vals)`: Computes product of `tanh` values for other variable nodes.
    - `prod_tanh = max(min(prod_tanh, 0.999999), -0.999999)`: Clips to avoid numerical overflow.
    - `msg_c2v(c, v) = 2 * atanh(prod_tanh)`: Computes check-to-variable message using inverse hyperbolic tangent.
  - **Total LLR and Codeword Estimation**:
    - `total_LLR(v) = LLR(v) + sum(msg_c2v(N{v}, v))`: Combines intrinsic LLR with all incoming check-to-variable messages.
    - `est_codeword = double(total_LLR < 0)`: Makes hard decision (0 or 1) based on sign of total LLR.
  - **Syndrome Check**:
    - `syn = mod(H * est_codeword', 2)`: Computes syndrome to check if `H * est_codeword^T = 0` (mod 2).
    - `if all(syn == 0)`: Terminates decoding early if the syndrome is zero (valid codeword).

---

### 12. Error Statistics
```
        %% === Error Statistics ===
        est_msg = est_codeword(1:k);
        bit_errors = sum(msg ~= est_msg);
        total_errors = total_errors + bit_errors;
        total_bits   = total_bits + k;
    end
    
    BER(idx) = total_errors / total_bits;
    fprintf('SNR = %d dB, BER = %.5f\n', SNR_dB, BER(idx));
end
```

- **Purpose**: Calculates the BER for the current SNR value.
  - `est_msg = est_codeword(1:k)`: Extracts the first `k` (32) bits of the estimated codeword as the decoded message.
  - `bit_errors = sum(msg ~= est_msg)`: Counts bit errors by comparing the original message with the decoded message.
  - `total_errors = total_errors + bit_errors`: Accumulates total bit errors across all blocks.
  - `total_bits = total_bits + k`: Accumulates total bits processed (k per block).
  - **After block loop**:
    - `BER(idx) = total_errors / total_bits`: Computes BER for the current SNR as the ratio of errors to total bits.
    - `fprintf('SNR = %d dB, BER = %.5f\n', SNR_dB, BER(idx))`: Prints SNR and corresponding BER for monitoring.

---

### 13. Plot BER Curve
```
%% === Plot BER curve ===
figure;
semilogy(SNR_dB_range, BER, '-o');
grid on;
xlabel('SNR (dB)');
ylabel('Bit Error Rate (BER)');
title('LDPC + QPSK + AWGN Simulation');
```

- **Purpose**: Visualizes the BER performance across the SNR range.
  - `figure`: Creates a new figure window.
  - `semilogy(SNR_dB_range, BER, '-o')`: Plots BER on a logarithmic y-axis against SNR, using a line with circle markers.
  - `grid on`: Adds a grid for better readability.
  - `xlabel`, `ylabel`, `title`: Labels the x-axis (SNR in dB), y-axis (BER), and sets the plot title.

---

### Output
<img width="281" height="164" alt="image" src="https://github.com/user-attachments/assets/d054e193-f4e3-4e48-8169-a49707592134" />

<img width="701" height="629" alt="image" src="https://github.com/user-attachments/assets/9c626dd1-7cc9-45ad-a086-0d27fc247d2a" />
