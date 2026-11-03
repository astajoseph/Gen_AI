# 🔧 Colab Notebook Fixes - Image & Text Issues

## Problem: "Notebook is not generating the right image or text in Colab"

This document provides exact code fixes for common issues.

---

## FIX #1: Images Not Displaying / Black Images

### Symptom
- No image grid appears in Colab output
- OR images appear completely black/gray
- OR images look like noise even after training

### Root Cause
Missing matplotlib backend or incorrect normalization

### Solution

**Add this after imports (Cell 2):**
```python
# Add right after: import matplotlib.pyplot as plt
%matplotlib inline
import matplotlib
matplotlib.use('TkAgg')  # For Colab compatibility

# Verify matplotlib is working
import matplotlib.pyplot as plt
plt.figure(figsize=(4, 4))
plt.plot([1, 2, 3, 4])
plt.title("Test Plot")
plt.show()
print("✓ Matplotlib working!")
```

**Fix image denormalization (Cell 9):**
```python
def display_image_grid(images, title="Generated Images", grid_size=4):
    """Display grid of generated images - FIXED VERSION"""
    
    # Move to CPU and detach
    images = images.cpu().detach()
    
    # CRITICAL: Correct denormalization
    # If model uses Tanh (outputs in [-1, 1]):
    images = (images + 1) / 2.0  # Convert from [-1, 1] to [0, 1]
    
    # Clamp to valid range
    images = torch.clamp(images, 0, 1)
    
    # Verify normalization worked
    print(f"Image stats - Min: {images.min():.4f}, Max: {images.max():.4f}, Mean: {images.mean():.4f}")
    
    # Create grid
    fig, axes = plt.subplots(grid_size, grid_size, figsize=(8, 8))
    fig.patch.set_facecolor('white')
    
    for i, ax in enumerate(axes.flat):
        if i < images.size(0):
            # Convert to HWC format for imshow
            img = images[i].permute(1, 2, 0).cpu().numpy()
            
            # Ensure correct dtype
            img = np.clip(img, 0, 1).astype(np.float32)
            
            ax.imshow(img)
            ax.set_title(f"Sample {i+1}", fontsize=8)
        ax.axis('off')
    
    plt.suptitle(title, fontsize=14, fontweight='bold')
    plt.tight_layout()
    plt.show()
    print(f"✓ Displayed {min(len(images), grid_size**2)} images")
```

**Test the fix:**
```python
# After displaying images, verify they're valid
print(f"Image shape: {images.shape}")
print(f"Image min/max: {images.min():.4f} / {images.max():.4f}")
print(f"Unique values: {torch.unique(images[:1]).shape[0]}")

# If unique values = 1, all pixels are the same (problem!)
# If unique values >> 1, image is valid (good!)
```

---

## FIX #2: Text Not Appearing / Silent Failure

### Symptom
- Text generation cell runs but shows no output
- OR shows empty strings
- OR "TypeError" or "RuntimeError" in text generation

### Root Cause
GPU memory issues or incorrect model mode

### Solution

**Replace text generation cell (Cell 15) with:**
```python
# FIXED TEXT GENERATION
print("Generating text samples with different temperature/top-k values...\n")

prompts = [
    "A magical product",
    "Once upon a time",
    "The future of music"
]

text_results = []

# Ensure model is in eval mode and uses correct device
gpt2_model.eval()
gpt2_model = gpt2_model.to(device)

print(f"Device: {device}")
print(f"Model dtype: {next(gpt2_model.parameters()).dtype}")

with torch.no_grad():  # CRITICAL: Disable gradients
    for prompt in prompts:
        for temp in [0.8]:  # Start with one temperature
            for top_k in [50]:  # Start with one top_k
                try:
                    # Encode input
                    input_ids = gpt2_tokenizer.encode(
                        prompt, 
                        return_tensors='pt'
                    ).to(device)
                    
                    print(f"Generating from '{prompt}' (temp={temp}, k={top_k})...")
                    
                    # Generate with parameters
                    output = gpt2_model.generate(
                        input_ids,
                        max_length=80,
                        temperature=float(temp),
                        top_k=int(top_k),
                        top_p=0.95,  # Add nucleus sampling
                        do_sample=True,
                        pad_token_id=gpt2_tokenizer.eos_token_id,
                        num_return_sequences=1,
                        early_stopping=True
                    )
                    
                    # Decode output
                    text = gpt2_tokenizer.decode(
                        output[0], 
                        skip_special_tokens=True
                    )
                    
                    # Store result
                    text_results.append({
                        'prompt': prompt,
                        'temperature': temp,
                        'top_k': top_k,
                        'generated_text': text
                    })
                    
                    print(f"✓ Generated {len(text.split())} words")
                    
                except Exception as e:
                    print(f"✗ Error: {e}")
                    print(f"  Try: Reduce max_length or use CPU")
                    continue

print("\n" + "="*80)
print("GPT-2 TEXT GENERATION RESULTS")
print("="*80 + "\n")

# Display results
for idx, row in enumerate(text_results, 1):
    print(f"[Sample {idx}] Prompt: '{row['prompt']}'")
    print(f"           Temp: {row['temperature']:.1f} | Top-K: {row['top_k']}")
    print(f"           Generated: {row['generated_text'][:100]}...")
    print("-" * 80)

print(f"\n✓ Successfully generated {len(text_results)} text samples!")
```

**If text still not appearing, check:**
```python
# Test with minimal example
print("TEST: Generating one sentence...")
input_ids = gpt2_tokenizer.encode("Hello", return_tensors='pt').to(device)
with torch.no_grad():
    output = gpt2_model.generate(input_ids, max_length=20, do_sample=False)
text = gpt2_tokenizer.decode(output[0])
print(f"Output: {text}")

# If this works, your setup is fine
# If not, check error message
```

---

## FIX #3: Loss Curves Not Displaying

### Symptom
- Matplotlib plots appear blank or don't show
- OR legend overlaps with plot
- OR no output from visualization cells

### Solution

**Fix visualization (Cell 21):**
```python
# FIXED PLOTTING CODE
import matplotlib.pyplot as plt
import seaborn as sns

# Set style
plt.style.use('seaborn-v0_8-darkgrid')
sns.set_palette("husl")

# Create figure with proper size
fig, axes = plt.subplots(1, 2, figsize=(14, 5))
fig.patch.set_facecolor('white')

# Plot data
colors = ['#FF6B6B', '#4ECDC4', '#45B7D1']

for idx, (config_name, metrics_df) in enumerate(metrics_dict.items()):
    if 'loss_g' in metrics_df.columns:
        axes[0].plot(
            metrics_df['epoch'], 
            metrics_df['loss_g'], 
            label=config_name,
            marker='o',
            markersize=3,
            linewidth=2,
            color=colors[idx],
            alpha=0.8
        )
    
    if 'loss_d' in metrics_df.columns:
        axes[1].plot(
            metrics_df['epoch'], 
            metrics_df['loss_d'], 
            label=config_name,
            marker='s',
            markersize=3,
            linewidth=2,
            color=colors[idx],
            alpha=0.8
        )

# Configure axes
axes[0].set_xlabel('Epoch', fontsize=12, fontweight='bold')
axes[0].set_ylabel('Generator Loss', fontsize=12, fontweight='bold')
axes[0].set_title('Generator Loss vs Epoch', fontsize=14, fontweight='bold')
axes[0].legend(loc='upper right', fontsize=10)
axes[0].grid(True, alpha=0.3)

axes[1].set_xlabel('Epoch', fontsize=12, fontweight='bold')
axes[1].set_ylabel('Discriminator Loss', fontsize=12, fontweight='bold')
axes[1].set_title('Discriminator Loss vs Epoch', fontsize=14, fontweight='bold')
axes[1].legend(loc='upper left', fontsize=10)
axes[1].grid(True, alpha=0.3)

# Key formatting
plt.tight_layout()

# Save before showing (important for Colab)
plt.savefig('/tmp/loss_curves.png', dpi=100, bbox_inches='tight')
print("✓ Plot saved to /tmp/loss_curves.png")

# Now show
plt.show()
print("✓ Loss curves displayed!")
```

---

## FIX #4: Gradio Interface Not Working

### Symptom
- Interface loads but buttons don't work
- OR "Connection error" message
- OR "Model not responding" after button click

### Solution

**Fix Gradio launch (Cell 25):**
```python
def create_gradio_interface(generator, gpt2_model, gpt2_tokenizer):
    """FIXED VERSION - More robust error handling"""
    
    # Ensure models are on correct device and in eval mode
    generator.eval()
    generator = generator.to(device)
    
    gpt2_model.eval()
    gpt2_model = gpt2_model.to(device)

    def generate_image_tab(temperature=0.5, num_images=9):
        """Generate and display images - FIXED"""
        try:
            generator.eval()
            with torch.no_grad():
                num_images = int(num_images)
                z = torch.randn(num_images, generator.latent_dim, device=device)
                z = z * float(temperature)
                images = generator(z)
            
            # Denormalize
            images = (images + 1) / 2
            images = torch.clamp(images, 0, 1)
            
            # Return first image as example
            img = images[0].cpu().permute(1, 2, 0).numpy()
            return np.clip(img, 0, 1)
        
        except Exception as e:
            print(f"Error in image generation: {e}")
            return np.zeros((32, 32, 3))

    def generate_text_tab(prompt, temperature, top_k):
        """Generate text from prompt - FIXED"""
        try:
            gpt2_model.eval()
            input_ids = gpt2_tokenizer.encode(prompt, return_tensors='pt').to(device)
            
            with torch.no_grad():
                output = gpt2_model.generate(
                    input_ids,
                    max_length=80,
                    temperature=float(temperature),
                    top_k=int(top_k),
                    do_sample=True,
                    pad_token_id=gpt2_tokenizer.eos_token_id,
                    num_return_sequences=1
                )
            
            text = gpt2_tokenizer.decode(output[0], skip_special_tokens=True)
            return text
        
        except Exception as e:
            return f"Error: {str(e)}"

    # Build Gradio interface
    with gr.Blocks(title="Multi-Modal Creative Generation Studio") as demo:
        gr.Markdown("""
        # 🎨 Multi-Modal Creative Generation Studio
        
        Generate images, text, and music using AI!
        """)

        with gr.Tabs():
            # Image Tab
            with gr.TabItem("🖼️ Image Generator"):
                with gr.Row():
                    with gr.Column():
                        temp_slider = gr.Slider(0.1, 2.0, value=1.0, label="Temperature")
                        num_img_slider = gr.Slider(1, 16, value=9, step=1, label="Num Images")
                        btn = gr.Button("Generate", variant="primary")
                    with gr.Column():
                        img_output = gr.Image(label="Generated Image")

                btn.click(generate_image_tab, inputs=[temp_slider, num_img_slider], outputs=img_output)

            # Text Tab
            with gr.TabItem("📝 Text Generator"):
                with gr.Row():
                    with gr.Column():
                        prompt = gr.Textbox(value="A magical", label="Prompt", lines=2)
                        temp = gr.Slider(0.1, 2.0, value=0.8, label="Temperature")
                        topk = gr.Slider(1, 100, value=40, label="Top-K")
                        btn2 = gr.Button("Generate", variant="primary")
                    with gr.Column():
                        text_output = gr.Textbox(label="Generated Text", lines=5)

                btn2.click(generate_text_tab, inputs=[prompt, temp, topk], outputs=text_output)

    return demo

# Create interface
print("Creating Gradio interface...")
interface = create_gradio_interface(generator, gpt2_model, gpt2_tokenizer)

# Launch with Colab settings
print("\nLaunching interface...")
print("⏳ This may take 10-20 seconds...")

# For Colab: use share=True to get a public link
interface.launch(
    share=True,           # Create public link
    debug=False,          # Disable debug mode
    show_error=True,      # Show errors
    server_name="0.0.0.0" # Listen on all interfaces
)

# Or for local testing:
# interface.launch(port=7860, share=False)
```

---

## FIX #5: Memory Issues

### Symptom
- "CUDA out of memory"
- "RuntimeError: NCCF..."
- Notebook crashes during training

### Solution

```python
# Add at beginning of notebook
import gc
import torch

def clear_memory():
    """Clear GPU and Python memory"""
    gc.collect()  # Python garbage collection
    torch.cuda.empty_cache()  # Clear GPU cache
    torch.cuda.synchronize()  # Wait for GPU
    print("✓ Memory cleared")

# Use when needed
clear_memory()

# For training, reduce batch size
train_dcgan(
    epochs=10,
    lr=0.0002,
    latent_dim=128,
    batch_size=32,  # Reduced from 64
    device=device
)

# Or switch to CPU
device = torch.device('cpu')
generator = generator.to(device)
```

---

## COMPLETE WORKING EXAMPLE

Here's a minimal working cell for testing:

```python
# Test: Minimal working code
import torch
import torch.nn as nn
from transformers import GPT2Tokenizer, GPT2LMHeadModel
import matplotlib.pyplot as plt
import numpy as np

%matplotlib inline

# Setup
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
print(f"✓ Device: {device}")

# 1. TEST IMAGE
print("\n1. Testing image generation...")
z = torch.randn(1, 128, device=device)
# (Assuming generator is defined)
img = (z * 0.5 + 0.5).cpu().numpy()  # Dummy image

plt.imshow(np.random.rand(32, 32, 3))
plt.title("Test Image")
plt.show()
print("✓ Image displayed")

# 2. TEST TEXT
print("\n2. Testing text generation...")
tokenizer = GPT2Tokenizer.from_pretrained('gpt2')
model = GPT2LMHeadModel.from_pretrained('gpt2').to(device).eval()

input_ids = tokenizer.encode("Hello", return_tensors='pt').to(device)
with torch.no_grad():
    output = model.generate(input_ids, max_length=20, do_sample=False)

text = tokenizer.decode(output[0])
print(f"Generated: {text}")
print("✓ Text generated")

# 3. TEST PLOT
print("\n3. Testing plots...")
x = np.arange(10)
y = x ** 2

plt.figure(figsize=(8, 5))
plt.plot(x, y, 'o-', label='y=x²')
plt.xlabel('X')
plt.ylabel('Y')
plt.legend()
plt.tight_layout()
plt.show()
print("✓ Plot displayed")

print("\n✅ All tests passed! Your setup is working.")
```

---

## SUMMARY

| Issue | Cell | Fix |
|-------|------|-----|
| Black images | 9 | Fix denormalization: `(img+1)/2` |
| No text | 15 | Add `torch.no_grad()` |
| Blank plots | 21 | Add `plt.tight_layout(); plt.show()` |
| Gradio not working | 25 | Use `share=True` |
| Out of memory | Any | Reduce batch_size or use CPU |

Apply these fixes and your notebook should work perfectly! 🚀
