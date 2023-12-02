# LLaVA-4V

## Brief

LLaVA-4V is a Large Vision-Language Model(LVLM) just like LLaVA and InstructBLIP. It is capable of chatting with a single image as context. To distinguish itself from others, its image-text datasets are generated by GPT4V. It does NOT not use any academic or text-only GPT4 pseudo image-text dataset. It should have similar performance to GPT4V for general purpose(not for benchmarking).

Its architecture is *indentical* to LLaVA-v1.5 but with better training data for image-caption alignment stage(Stage 1) and instruction fine-tuning stage(Stage 2).

After several months of researching on the hallucination of LVLMs, my efforts are doomed due to the release of GPT4V and its API. After tuning a simple model with finer data from GPT4V, the hallucinations are just gone, needless for any tricky methods. This model serves as a goodbye letter and a reminder to contemporary researchers that data quality really, really matters.

## Usage

The architure is identical to LLaVA-v1.5, so the {train, eval, demo, serve} are all the same as LLaVA-v1.5. See

[https://github.com/haotian-liu/LLaVA](https://github.com/haotian-liu/LLaVA)

Then specifying the models in the following model zoo.

## Note

The training is done on 8x80G gpus on 1 node, the settings are the same as llava-v1.5 unless stated otherwise.

## Model Zoo

- For stage-1 training, we use deepspeed zero2 by default. But for 13b we use zero3 and `gradient_accumulate_steps=2` and half `per_device_batch_size` to avoid OOM.

| Name                                | Comment                          | Stage | Checkpoint                                                   | LLM             | Vision Encoder | Projection |
| ----------------------------------- | -------------------------------- | ----- | ------------------------------------------------------------ | --------------- | -------------- | ---------- |
| LLaVA-4V-13B_vit-l14-336px          | **best for general use**         | 1,2   | training                                                     | Vicuna-13B-v1.5 | CLIP-L14-336px | MLP-2x     |
| LLaVA-4V-13B_vit-l14-336px_Pretrain | best for captioning              | 1     | training                                                     | Vicuna-13B-v1.5 | CLIP-L14-336px | MLP-2x     |
| LLaVA-4V-7B_vit-l14-336px           | good for general use             | 1,2   | [Maxlinn/LLaVA-4V-7B_vit-l14-336px](https://huggingface.co/Maxlinn/LLaVA-4V-7B_vit-l14-336px) | Vicuna-7B-v1.5  | CLIP-L14-336px | MLP-2x     |
| LLaVA-4V-7B_vit-l14-336px_Pretrain  | good for captioning              | 1     | [Maxlinn/LLaVA-4V-7B_vit-l14-336px_Pretrain](https://huggingface.co/Maxlinn/LLaVA-4V-7B_vit-l14-336px_Pretrain) | Vicuna-7B-v1.5  | CLIP-L14-336px | MLP-2x     |
| LLaVA-4V-7B_vit-b32                 | most lightweight for general use | 1,2   | training                                                     | Vicuna-7B-v1.5  | CLIP-B32       | MLP-2x     |
| LLaVA-4V-7B_vit-b32_Pretrain        | most lightweight for captioning  | 1     | training                                                     | Vicuna-7B-v1.5  | CLIP-B32       | MLP-2x     |


## Training

The procedure is similar to LLaVA-v1.5 but with different dataset.

### Dataset Overview

| Name                      | Stage | Pretrain  Data Amount | Finetune Data Amount        | Pretrain Data                                                | Finetune Data                                                |
| ------------------------- | ----- | --------------------- | --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| any general purpose model | 1,2   | 1,166,048             | 359,783<br>=96384+222711+40688 | share-captioner_coco_lcs_sam_1246k_1107.json(filtered ill examples) | sharegpt4v_instruct_gpt4-vision_cap100k.json(filtered ill and non-exist examples)<br>lvis_instruct4v_220k.json<br>llava_v1_5_mix665k.json(only text-only examples) |
| any pretrain model        | 1     | 1,166,048             | /                           | share-captioner_coco_lcs_sam_1246k_1107.json(filtered ill examples) | /                                                            |


### Stage 1: image-caption alignment

In this stage, only the projector `mlp2x_gelu` is trained. After training, it could do captioning in LLaVA's `plain` template (i.e. no chat template).

Dataset used:
- https://huggingface.co/datasets/Lin-Chen/ShareGPT4V, share-captioner_coco_lcs_sam_1246k_1107.json
    - It is NOT directly from GPT4V. It is a collection of caption data generated by Share-Captioner, which is a captioner trained on GPT4V captioning data. You can find more information in the huggingface page.
    - NOTE: we removed 80853 ill examples from it which contains 'sa_', see examples at #Ill examples of ShareGPT4V.
    - Contains images from
        - coco train2017
        - llava pretrain
        - sam pretrain dataset 000000-000051


### Stage 2: instruction fine-tuning

In this stage, both the projector and the LLM are trained.

Dataset used:
- https://huggingface.co/datasets/Lin-Chen/ShareGPT4V, sharegpt4v_instruct_gpt4-vision_cap100k.json
    - Pure golden captioning instructions by prompting GPT4V.
    - NOTE: we removed 5637 ill examples from it which contains 'sa_', see examples at #Ill examples of ShareGPT4V.
    - NOTE: we removed 4 examples which 'image' is not exist in their provided images. see at #Non-exist examples of ShareGPT4V.
    - Contains images from
        - coco train2017
        - llava pretrain
        - sam pretrain dataset 000000-000051
        - share_textvqa
        - web-celebrity
        - web-landmark
        - wikiart
- https://huggingface.co/datasets/X2FD/LVIS-Instruct4V, lvis_instruct4v_220k.json
    - Pure golden diverse instructions by prompting GPT4V.
    - Contains images from
        - coco train2017
        - coco val2017
- https://huggingface.co/datasets/liuhaotian/LLaVA-Instruct-150K/blob/main/llava_v1_5_mix665k.json
    - NOTE: we only used text-only 40688 multi-turn chat examples from it to maintain text-only performance.


## Appendix

### Credits

We extend our thanks to the team behind the datasets, you can find their homepages here:
- ShareGPT4V: https://sharegpt4v.github.io
- LVIS: https://github.com/X2FD/LVIS-INSTRUCT4V
- LLaVA: https://github.com/haotian-liu/LLaVA

### License

Attribution-NonCommercial 4.0 International It should abide by the policy of OpenAI: https://openai.com/policies/terms-of-use

### Ill examples of ShareGPT4V

```py
{'id': 'sa_518556',
 'image': 'sam/images/sa_518556.jpg',
 'conversations': [{'from': 'human',
   'value': "<image>\nWhat's happening in the scene?"},
  {'from': 'gpt',
   'value': "The image captures a charming scene of a two-story brick house nestled on a street corner. The house, with its black roof and chimney, exudes a sense of warmth and homeliness. A red car is parked in front of the house, adding a pop of color to the scene. On the left side of the house, a tree stands tall, providing a touch of nature amidst the urban setting. The sky above is a clear blue, suggesting a bright and sunny day. The perspective of the image is from the street, looking up at the house, giving a sense of scale and grandeur to the structure. The image does not provide any information that can be used to identify the landmark as 'sa_17448'."}]}

{'id': 'sa_164019',
 'image': 'sam/images/sa_164019.jpg',
 'conversations': [{'from': 'human',
   'value': "<image>\nWhat is this photo about'?"},
  {'from': 'gpt',
   'value': 'The image captures a serene scene of a mountain lake. The lake, with its light blue-green hue, is nestled in the heart of a rocky landscape. The rocks, a mix of gray and brown, are scattered around the lake, adding a rugged charm to the scene. The perspective of the image is from above, providing a bird\'s eye view of the lake and its surroundings. The sky, peeking through the top left corner of the image, adds a sense of depth and vastness to the scene. The image does not provide any specific information about the landmark "sa_17448". However, the unique combination of natural elements and the tranquil setting might be recognizable to those familiar with the area.'}]}

...
```

### Non-exist examples of ShareGPT4V

```py
[{'id': 'Choi_Min-sik',
  'image': 'web-celebrity/images/Choi_Min-sik.jpg',
  'conversations': [{'from': 'human',
    'value': 'What do you see happening in this image?\n<image>'},
   {'from': 'gpt',
    'value': 'In the image, renowned actor Choi Min-sik is captured in a scene from a movie. He is seated in a red chair, located in a dimly lit room. Dressed in a formal black suit and tie, he holds a book in his hand, suggesting he might be reading or about to read. His face bears a serious expression, indicating the intensity of the scene. The background is slightly blurred, but a shelf filled with various objects can be discerned, adding depth to the setting.'}]},
 {'id': 'Choi_Min-sik2',
  'image': 'web-celebrity/images/Choi_Min-sik2.jpg',
  'conversations': [{'from': 'human',
    'value': 'Analyze the image in a comprehensive and detailed manner.\n<image>'},
   {'from': 'gpt',
    'value': 'In this image, the renowned actor Choi Min-sik is captured in a moment of quiet contemplation. Seated in a chair against the backdrop of a brick wall, he is adjusting his glasses with his right hand, a serious expression etched on his face. His attire, a gray suit, adds to the solemnity of the scene. To his side, a desk holds a lamp with a yellow shade adorned with a geometric pattern, casting a soft glow. A plaque resting on the desk reads "Mechanical Properties of Materials", hinting at the intellectual pursuits of the character Choi Min-sik might be portraying.'}]},
 {'id': 'Lee_Byung-hun',
  'image': 'web-celebrity/images/Lee_Byung-hun.jpg',
  'conversations': [{'from': 'human',
    'value': 'Write a detailed description of the given image.\n<image>'},
   {'from': 'gpt',
    'value': "In this image, the renowned actor Lee Byung-hun is captured in a moment of deep contemplation. He is seated in a public transportation vehicle, possibly a bus or train. Dressed in a crisp white shirt and a blue jacket, he gazes intently to the left, his serious expression suggesting a moment of intense thought or concern. The background is a blur of other passengers, their faces and details obscured, emphasizing the actor's presence in the foreground. The overall color palette of the image leans towards shades of blue and gray, adding to the somber mood of the scene."}]},
 {'id': 'Lee_Byung-hun2',
  'image': 'web-celebrity/images/Lee_Byung-hun2.jpg',
  'conversations': [{'from': 'human',
    'value': "What is this photo about'?\n<image>"},
   {'from': 'gpt',
    'value': "This image is a vibrant collage featuring the versatile actor Lee Byung-hun in three distinct roles. On the left, Lee Byung-hun commands attention in a black military uniform adorned with gold epaulettes and a striking red collar, embodying an air of authority. In the center, he exudes a sense of mystery, donning a brown jacket and casting a serious gaze off to the side. On the right, he appears in a gold hooded jacket and a mask, adding an element of intrigue. The collage is set against a pale peach background, punctuated with playful orange and yellow circles and lines, highlighting the actor's diverse roles."}]}]
```
