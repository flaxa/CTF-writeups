# TCP1P 2024 - Imperfect Guesser

- CTF: TCP1P 2024
- Challenge Name: Imperfect Guesser
- Category: Reverse
- Difficulty: Unknown

## Synopsis

Use gradient descent to reverse-engineer a PyTorch model, aiming to recover the input from a given output to retrieve a hidden flag

## Description

Leaving a floating secret crumbs with the help of untrained labour, now "guess" the flag.

## Challenge

The challenge contains one file:

- `chally.py`: This is the main file that hides the flag by passing it through a PyTorch model

```python
import torch, random
import torch.nn
import numpy as np

flag = "TCP1P{REDACTED}"

def floatify(ip):
	flag = [float(ord(i)) for i in ip]
	normalized = torch.tensor([flag], dtype=torch.float32)
	return normalized

def tf(_in,_out):
	weight = np.round(np.random.uniform(-1, 1, (_out, _in)).astype(np.float32),2)
	bias = np.round(np.random.uniform(-1, 1, _out).astype(np.float32),2)
	return torch.from_numpy(weight), torch.from_numpy(bias)

np.random.seed(0x544350)
model = torch.nn.Sequential(
	torch.nn.Linear(24, 450),
	torch.nn.Linear(450, 128),
	torch.nn.Linear(128, 17)
)

layer_shapes = [(24, 450), (450, 128), (128, 17)]

for i, (input_dim, output_dim) in enumerate(layer_shapes):
	weight, bias = tf(input_dim, output_dim)
	model[i].weight.data = weight
	model[i].bias.data = bias
 
print([i.detach().numpy().tolist() for i in model(floatify(flag))[0]])
# Output:
# [38883.9140625, 18747.87890625, -15371.05078125, 12231.2080078125, -56379.48046875, -33719.13671875, 9454.150390625, 9346.9814453125, 1701.4693603515625, -6380.3759765625, 12019.501953125, -4850.94140625, 14421.296875, 44332.0390625, -11196.283203125, -19712.0859375, -36390.265625]
```

## Solution

### Problem Setup

In this scenario, the model has been trained to map a 24-character input (representing the flag) to a 17-dimensional output. The challenge is to reverse the process: given the output, recover the original input (the flag).

The flag format follows a known structure:

- The first six characters are `"TCP1P{"`.
- The last character is `"}"`.
- The remaining 17 characters need to be inferred.

### Step 1: Target Output Setup

First, we define the target output, which is the known output from the model for a specific input.

```python
output = [38883.9140625, 18747.87890625, -15371.05078125, 12231.2080078125, -56379.48046875,
          -33719.13671875, 9454.150390625, 9346.9814453125, 1701.4693603515625, -6380.3759765625,
          12019.501953125, -4850.94140625, 14421.296875, 44332.0390625, -11196.283203125,
          -19712.0859375, -36390.265625]

# Convert the output into a format PyTorch can use
target_output = torch.tensor([output], dtype=torch.float32)
```

The `output` list represents the expected result from the neural network. We convert it into a PyTorch tensor so it can be used as a target for optimisation.

### Step 2: Constructing an Initial Flag Candidate

We know the first six and last characters of the flag, but the 17 characters in the middle are unknown. We start by randomly initialising these 17 characters.

```python
# Known characters of the flag
first_six_chars = "TCP1P{"  # Known prefix
last_char = "}"              # Known suffix

# Construct the initial flag candidate
known_chars = [ord(c) for c in first_six_chars]
middle_length = 17  # Total length is 24; 6 + middle + 1 = 24, so 17 middle characters
middle_chars = np.random.randint(32, 128, size=(middle_length)).tolist()  # Random initial middle characters
known_chars += middle_chars + [ord(last_char)]

# Create an input tensor with the known characters
input_tensor = torch.tensor([known_chars], dtype=torch.float32, requires_grad=True)
```

Here, we initialise the middle part of the flag randomly and combine it with the known prefix and suffix. The complete flag candidate is stored in a PyTorch tensor, and gradients are enabled for optimisation.

### Step 3: Setting Up the Optimiser

We use the Adam optimizer to iteratively adjust the input tensor to minimize the difference between the model's output for the current flag candidate and the target output.

```python
# Optimiser setup
optimiser = torch.optim.Adam([input_tensor], lr=0.1)  # Learning rate
```

The learning rate controls the step size during optimisation. Setting it to 0.1 allows for gradual adjustments.

### Step 4: Gradient Descent Optimisation

The goal of this step is to iteratively refine the input tensor so that the model's output matches the target output as closely as possible. This is done by minimising the mean squared error (MSE) loss.

```python
# Number of iterations for optimisation
num_iterations = 50000

# Gradient-based optimisation loop
for iteration in range(num_iterations):
    optimiser.zero_grad()  # Zero out the gradients

    # Forward pass through the model
    output = model(input_tensor)

    # Compute the loss
    loss = torch.nn.functional.mse_loss(output, target_output)
    
    # If loss is zero break from the training loop
    if loss.item() == 0.0:
        break

    # Backward pass (compute gradients)
    loss.backward()

    # Gradient update step
    optimiser.step()

    with torch.no_grad():
        # Ensure the first six and last character are fixed
        input_tensor[0, :6] = torch.tensor([ord(c) for c in first_six_chars], dtype=torch.float32)
        input_tensor[0, -1] = ord(last_char)

    # Print progress
    if iteration % 100 == 0:
        print(f"Iteration {iteration}, Loss: {loss.item()}")
```

- The optimiser updates the input tensor based on the gradients of the loss function with respect to the input.
- To maintain the known structure of the flag, the first six and last characters are kept fixed after each update.
- The process continues for a maximum of 50,000 iterations or until the loss reaches zero.

### Step 5: Converting the Optimised Tensor Back to a Flag

Finally, we convert the adjusted input tensor into a string that represents the optimised flag candidate.

```python
# Convert the optimised input tensor back to a flag candidate
optimised_flag = "".join(chr(int(round(c))) for c in input_tensor.detach().numpy()[0])

print(f"Optimised Flag Candidate: {optimised_flag}")
```

After the optimisation loop, we round the values in the tensor to the nearest integers and convert them to characters, forming the recovered flag.

### Summary

This code attempts to reverse the neural network's behaviour by finding an input (flag) that produces a given output. It leverages gradient descent to minimise the error between the model's output for the current input and the target output, adjusting the input iteratively while keeping certain known characters fixed. The process is complete when the loss reaches zero, indicating successful flag recovery.

### Output

```
Iteration 0, Loss: 132506472.0
Iteration 100, Loss: 84305080.0
Iteration 200, Loss: 51630704.0
Iteration 300, Loss: 30629590.0
...
Iteration 49900, Loss: 0.006987122818827629
Optimised Flag Candidate: TCP1P{1ts_tr3ndy_NN_n0w}
```

## Solution Code

```python
import torch, random
import torch.nn
import numpy as np

flag = "TCP1P{}"

def floatify(ip):
	flag = [float(ord(i)) for i in ip]
	normalized = torch.tensor([flag], dtype=torch.float32)
    
	return normalized
def tf(_in,_out):
	weight = np.round(np.random.uniform(-1, 1, (_out, _in)).astype(np.float32),2)
	bias = np.round(np.random.uniform(-1, 1, _out).astype(np.float32),2)
	return torch.from_numpy(weight), torch.from_numpy(bias)

np.random.seed(0x544350)
model = torch.nn.Sequential(
	torch.nn.Linear(24, 450),
	torch.nn.Linear(450, 128),
	torch.nn.Linear(128, 17)
)


layer_shapes = [(24, 450), (450, 128), (128, 17)]

for i, (input_dim, output_dim) in enumerate(layer_shapes):
	weight, bias = tf(input_dim, output_dim)
	model[i].weight.data = weight
	model[i].bias.data = bias
 

# Output:
# [38883.9140625, 18747.87890625, -15371.05078125, 12231.2080078125, -56379.48046875, -33719.13671875, 9454.150390625, 9346.9814453125, 1701.4693603515625, -6380.3759765625, 12019.501953125, -4850.94140625, 14421.296875, 44332.0390625, -11196.283203125, -19712.0859375, -36390.265625]

output = [38883.9140625, 18747.87890625, -15371.05078125, 12231.2080078125, -56379.48046875, -33719.13671875, 9454.150390625, 9346.9814453125, 1701.4693603515625, -6380.3759765625, 12019.501953125, -4850.94140625, 14421.296875, 44332.0390625, -11196.283203125, -19712.0859375, -36390.265625]

#convert the output into a format PyTorch can use
target_output = torch.tensor([output], dtype=torch.float32)


# Known characters of the flag
first_six_chars = "TCP1P{"  # Known prefix
last_char = "}"              # Known suffix

# Construct the initial flag candidate
known_chars = [ord(c) for c in first_six_chars]
middle_length = 17  # Total length is 24; 6 + middle + 1 = 24, so 17 middle characters
middle_chars = np.random.randint(32, 127, size=(middle_length)).tolist()  # Random initial middle characters
known_chars += middle_chars + [ord(last_char)]

# Create an input tensor with the known characters
input_tensor = torch.tensor([known_chars], dtype=torch.float32, requires_grad=True)

# Target output (adjust according to your expectations)

# Optimiser setup
optimiser = torch.optim.Adam([input_tensor], lr=0.1)  # Learning rate

# Number of iterations for optimisation
num_iterations = 50000

# Gradient-based optimisation loop
for iteration in range(num_iterations):
    optimiser.zero_grad()  # Zero out the gradients

    # Forward pass through the model
    output = model(input_tensor)

    # Compute the loss
    loss = torch.nn.functional.mse_loss(output, target_output)
    
    # If loss is zero break from training loop
    if loss.item() == 0.0:
        break

    # Backward pass (compute gradients)
    loss.backward()

    # Gradient update step
    optimiser.step()

    with torch.no_grad():

        # Ensure the first six and last character are fixed
        input_tensor[0, :6] = torch.tensor([ord(c) for c in first_six_chars], dtype=torch.float32)
        input_tensor[0, -1] = ord(last_char)

    #print progress
    if iteration % 100 == 0:
        print(f"Iteration {iteration}, Loss: {loss.item()}")
    

# Convert the optimised input tensor back to a flag candidate
optimised_flag = "".join(chr(int(round(c))) for c in input_tensor.detach().numpy()[0])

print(f"Optimised Flag Candidate: {optimised_flag}")
```

### [Home](/)
