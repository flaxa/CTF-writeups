# <img style="vertical-align:middle" src="/Assets/flaxa_zoom.png" width="30"  alt=""/> TCP1P 2024 - Skibidi Format

- CTF: TCP1P 2024
- Challenge Name: Skibidi Format
- Category: Forensics
- Difficulty: Unknown


## Synopsis

The solution involves reading a `.skibidi` file, extracting the header metadata, decrypting the compressed pixel data using AES-256-GCM, and decompressing it with Zstandard. The pixel data is then reconstructed into an image using the `PIL` library, based on the specified number of colour channels. 

## Description

So my friend just made a new image format and asked me to give him a test file, so I gave him my favorite png of all time. But the only thing I receive back is just my image with his new format and its "specification" file, don't know what that is. Can you help me read this file?

## Challenge

The challenge contains two files:

- `spec.html`: This is a file describing the detail of the new photo format
- `suisei.skibidi` This is the picture in the format specified by the `spec.html` file

### File Format Specification 
Based on the `spec.html` a `.skibidi` file consists of two main sections: 
1. **Header**: Contains metadata about the image, such as dimensions, colour channels, compression method, and encryption details. 
2. **Data Section**: Holds the encrypted and compressed pixel data. 
#### Header Structure 

The header has a fixed size of 58 bytes and includes the following fields:

| Offset | Field        | Size (bytes) | Description                             |   
| ------ | ------------ | ------------ | --------------------------------------- | 
| 0      | Magic Number | 4            | Identifier ("SKB1") for the file format |     
| 4   | Width | 4   | Image width in pixels |    
| 8   | Height | 4   | Image height in pixels |   
| 12  | Color Channels | 1   | Number of color channels (1 = Grayscale, 3 = RGB, 4 = RGBA) | 
| 13  | Compression ID | 1   | Compression algorithm identifier (1 = Zstandard) |   
| 14  | AES Key | 32  | AES-256 encryption key |    
| 46  | AES Initialization Vector (IV) | 12  | AES-GCM IV |  |
#### Data Section 
The pixel data is encrypted and compressed, following these steps: 
1. **Compression**: Pixel data is compressed using the specified compression algorithm (currently, Zstandard). 
2. **Encryption**: Compressed data is encrypted using AES-256-GCM with the provided key and IV.

## Solution
To solve this problem, we need to: 
1. Read the `.skibidi` file, extracting header information. 
2. Decrypt the encrypted data using the AES key and IV. 
3. Decompress the decrypted data using the specified compression method. 
4. Reconstruct the image using the decompressed pixel data.

### Step 1. Importing Required Libraries 

The necessary libraries are imported, including `struct` for reading binary data, `Cryptodome.Cipher` for AES decryption, `zstandard` for decompression, and `PIL` and `numpy` for image manipulation. 

```python 
import struct
from Cryptodome.Cipher import AES
import zstandard as zstd
from PIL import Image
import numpy as np
```

### Step 2. Reading the `.skibidi` File

The `read_skibidi_file` function takes the file path of a `.skibidi` file, opens it in binary mode, and reads the header to extract metadata such as image dimensions, colour channels, compression method, and encryption details.

```python
def read_skibidi_file(file_path):
    with open(file_path, 'rb') as f:
        # Read header
        magic = f.read(4)
        if magic != b'SKB1':
            raise ValueError("Invalid file format")
        
        width = struct.unpack('<I', f.read(4))[0]
        height = struct.unpack('<I', f.read(4))[0]
        channels = struct.unpack('B', f.read(1))[0]
        compression_id = struct.unpack('B', f.read(1))[0]
        
        aes_key = f.read(32)
        aes_iv = f.read(12)
        
        # Read encrypted data
        encrypted_data = f.read()
        ...

```

When running this code and printing out the values for width, height, channels and compression_id you get the following output :

```
Width 3840 - height 2160 - channels  4 - compression 1
```

These values will be useful in the future when it comes to converting the data into an image, the compression method is also using Zstandard which is the only method supported based on the specification.

### Step 3. Decrypting the Pixel Data

Using the AES-256-GCM algorithm, the encrypted pixel data is decrypted with the extracted key and IV from the header. This step ensures data confidentiality and integrity.

```python
		...
        # Decrypt the data
        cipher = AES.new(aes_key, AES.MODE_GCM, nonce=aes_iv)
        decrypted_data = cipher.decrypt(encrypted_data)
        ...

```

### Step 4. Decompressing the Data

The decompressed pixel data is obtained using the Zstandard library, based on the specified compression method in the header. Error handling ensures that decompression issues are reported.

```python
		...
        # Decompress the data
        try:
            dctx = zstd.ZstdDecompressor()
            pixel_data = dctx.decompress(decrypted_data, max_output_size=100000000)
        except zstd.ZstdError as e:
            print("Decompression failed:", e)
            return None
        except Exception as e:
            print("An error occurred:", e)
            return None
        
        # Return the extracted data
        return width, height, channels, compression_id, pixel_data
```

### Step 5. Reconstructing the Image

The image is reconstructed from the decompressed pixel data. Since the number of channels is 4, the pixel data is reshaped accordingly. The `PIL` library is used to convert the array into an image and save or display it.

```python
width, height, channels, compression, pixel_data = read_skibidi_file("./suisei.skibidi")

print(f"Width {width} - height {height} - channels  {channels} - compression {compression}")

width = 3840  
height = 2160  
rgba_data = pixel_data 

# Convert byte array to a NumPy array
image_array = np.frombuffer(rgba_data, dtype=np.uint8).reshape((height, width, 4))

# Create an image from the array
image = Image.fromarray(image_array, 'RGBA')

# Save or display the image
image.save('output_image.png')
image.show()
```

### Summary

The solution shows how to read the `.skibidi` file format by:

1. Extracting metadata from the header.
2. Decrypting and decompressing the pixel data.
3. Reconstructing the image using `PIL` based on the colour channels.

### Output

The output for this code is an image where the photo contains the flag.

![](images/output_image.png)

## Solution Code

```python
import struct
from Cryptodome.Cipher import AES
import zstandard as zstd
from PIL import Image
import numpy as np

def read_skibidi_file(file_path):
    with open(file_path, 'rb') as f:
        # Read header
        magic = f.read(4)
        if magic != b'SKB1':
            raise ValueError("Invalid file format")
        
        width = struct.unpack('<I', f.read(4))[0]
        height = struct.unpack('<I', f.read(4))[0]
        channels = struct.unpack('B', f.read(1))[0]
        compression_id = struct.unpack('B', f.read(1))[0]
        
        aes_key = f.read(32)
        aes_iv = f.read(12)
        
        # Read encrypted data
        encrypted_data = f.read()

        # Decrypt the data
        cipher = AES.new(aes_key, AES.MODE_GCM, nonce=aes_iv)
        decrypted_data = cipher.decrypt(encrypted_data)
        print(len(decrypted_data))
        print(decrypted_data[0:10])
        # Decompress the data
        try:
            dctx = zstd.ZstdDecompressor()
            pixel_data = dctx.decompress(decrypted_data, max_output_size=100000000)
        except zstd.ZstdError as e:
            print("Decompression failed:", e)
        except Exception as e:
            print("An error occurred:", e)
        
        # Interpret pixel data based on channels
        return width, height, channels, compression_id, pixel_data

width, height, channels, compression, pixel_data = read_skibidi_file("./suisei.skibidi")
print(f"Width {width} - height {height} - channels  {channels} - compression {compression}")

width = 3840  
height = 2160  
rgba_data = pixel_data 

# Convert byte array to a NumPy array
image_array = np.frombuffer(rgba_data, dtype=np.uint8).reshape((height, width, 4))

# Create an image from the array
image = Image.fromarray(image_array, 'RGBA')

# Save or display the image
image.save('output_image.png')
image.show()
```

---
### [Home](/)
