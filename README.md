# Detect Dominant Colors in Frames of Video

This is a early pass at having an understanding of how to measure the dominant colors of a video. The idea came to me during a job interview and I wanted to see if I could get it working quickly. It's a little rough, especially around temporarily creating a directory, writing files to it, reading those files and then removing that directory, but it's the execution of the initial idea.

The output is:
- a gif of the color palate for each frame of video, the number of colors is variable
- a png of the dominant color for each frame
- a png of the average color for each frame

## Exmaples

Here's a video from the Museum of Ice Cream's Instagram account: https://www.instagram.com/p/B3IHhXZgz2Y/

Here's the palatte gif for that video, which shows the ten five colors used and their representation in each frame:

![Palatte gif](https://raw.githubusercontent.com/robblatt/dominant-colors-of-video/master/output/sample_video_2_palattes.gif)

The PNG showing the average color for each frame. The colors shown are average RGB values, so they aren't the actual colors in the video. It's not useful for the example of what I'm looking to accomplish, but not everything we attempt will always be:

![Average color per frame](https://raw.githubusercontent.com/robblatt/dominant-colors-of-video/master/output/sample_video_2_avg_color.png)

The PNG showing the dominant color for each frame.

![Average color per frame](https://raw.githubusercontent.com/robblatt/dominant-colors-of-video/master/output/sample_video_2_dom_color.png)

## Code

Here's the .py file for the function. The notebook file is set to output to a testing/ directory so the images don't get confused.

```
import cv2
import numpy as np
from skimage import io
import glob
import pandas as pd
from PIL import Image
import imageio
import os
import shutil
import matplotlib.pyplot as plt
from tqdm import tqdm
import math

print('Video file:')
user_input_file_name = input()

print('Number of colors to include in gif palatte:')
user_input_colors_for_palatte = input()

def dom_avg_video(file_name, total_colors):
    path = "images/vid_cap/"
    try:
        os.makedirs(path)
    except OSError:
        print ("Creation of the temp directory %s failed" % path)
    else:
        print ("Successfully created the temp directory %s " % path)

    print('writing frames')
    cap= cv2.VideoCapture(file_name)
    i=0
    while(cap.isOpened()):
        ret, frame = cap.read()
        if ret == False:
            break
        if i < 10:
            cv2.imwrite(path + '000' + str(i) + '.png',frame)
        elif i < 100:
            cv2.imwrite(path + '00' + str(i) + '.png',frame)
        elif i < 1000:
            cv2.imwrite(path + '0' + str(i) + '.png',frame)
        else:
            cv2.imwrite(path + str(i) + '.png',frame)
        i+=1

    cap.release()
    cv2.destroyAllWindows()
    
    save_path = 'output/' + file_name.split('/')[-1].split('.')[0]

    frame_path = "images/vid_frame/"
    try:
        os.makedirs(frame_path)
    except OSError:
        print ("Creation of the temp directory %s failed" % frame_path)
    else:
        print ("Successfully created the temp directory %s " % frame_path)


    print('analyzing dominant color palatte')

    
    dominant_colors = {}
    average_colors = {}

    filelist = sorted(glob.glob(path + '*.png'))

    j = 0

    for filepath in tqdm(filelist):
        img = io.imread(filepath) #[:, :, :-1]

        pixels = np.float32(img.reshape(-1, 3))

        n_colors = int(total_colors)
        criteria = (cv2.TERM_CRITERIA_EPS + cv2.TERM_CRITERIA_MAX_ITER, 200, .1)
        flags = cv2.KMEANS_RANDOM_CENTERS

        _, labels, palette = cv2.kmeans(pixels, n_colors, None, criteria, 10, flags)
        _, counts = np.unique(labels, return_counts=True)

        dominant = palette[np.argmax(counts)]

        myimg = cv2.imread(filepath)
        avg_color_per_row = np.average(myimg, axis=0)
        avg_color = np.average(avg_color_per_row, axis=0)

        dominant_colors[filepath[8:].split('.')[0]] = dominant
        average_colors[filepath[8:].split('.')[0]] = avg_color

        indices = np.argsort(counts)[::-1]   
        freqs = np.cumsum(np.hstack([[0], counts[indices]/counts.sum()]))
        rows = np.int_(img.shape[0]*freqs)

        dom_patch = np.zeros(shape=img.shape, dtype=np.uint8)
        for i in range(len(rows) - 1):
            dom_patch[rows[i]:rows[i + 1], :, :] += np.uint8(palette[indices[i]])

        fig, (ax0) = plt.subplots(1, 1, figsize=(5,5))
        ax0.imshow(dom_patch)
        ax0.set_title('Dominant colors of frame ' + filepath.split('/')[-1].split('.')[0])
        ax0.axis('off')

        available_length = len(str(len(filelist))) - len(str(j))
        fig.savefig(frame_path + 'dom_colors_pyplot_' + str(available_length * '0') + str(j) + '.png')
        
        j += 1

        plt.close()

    print('writing palatte gif')

    filenames = sorted(glob.glob(frame_path + '*.png'))

    images = []
    for filename in filenames:
        images.append(imageio.imread(filename))
    imageio.mimsave(save_path + '_palattes.gif', images)

    df_d = pd.DataFrame.from_dict(dominant_colors).transpose()
    df_a = pd.DataFrame.from_dict(average_colors).transpose()

    df_d.reset_index(inplace = True, drop = True)
    df_a.reset_index(inplace = True, drop = True)

    df_a = df_a.round(0).astype(int)
    df_d = df_d.round(0).astype(int)

    remove_path = 'images/'
    try:
        shutil.rmtree(remove_path)
    except OSError:
        print ("Deletion of the temp directory %s failed" % remove_path)
    else:
        print ("Successfully deleted the temp directory %s" % remove_path)

    print('writing dominant and average color charts')
    
    width = 500
    height = math.ceil((len(df_d) / width) * 10 ) * 10

    im = Image.new('RGB', (width,height))

    for i in range(len(df_d)):
        for j in range(1,9):
            for k in range (1,9):
                x = (j + (i * 10)) % width
                y = (j + (i * 10)) // width
        #         print(x,y)
                colors = (df_d[0][i], df_d[1][i], df_d[2][i])
                im.putpixel((x,(k + (y * 10))), colors)

    im.save(save_path + '_dom_color.png')

    im = Image.new('RGB', (width,height))

    for i in range(len(df_a)):
        for j in range(1,9):
            for k in range (1,9):
                x = (j + (i * 10)) % width
                y = (j + (i * 10)) // width
        #         print(x,y)
                colors = (df_a[0][i], df_a[1][i], df_a[2][i])
                im.putpixel((x,(k + (y * 10))), colors)

    im.save(save_path + '_avg_color.png')
    
    print('Complete')

dom_avg_video(user_input_file_name, user_input_colors_for_palatte)
```

## Moving Forward

I'm not sure if I'll be coming back to make substantial updates to this repository, but if I do it will be likely be these items:
- speed up the processing of the color palatte files
  - depending on the number of colors selected, it could take 5-60x the length of the video to process
- use something that isn't pandas to to create the dominant and average color images
- check the file path and color number user input and display an error
