### I/O data structure

```apex

inputs:# 33 dict
	#K  4scale
	K,0
	K,1
	K,2
	K,3
	#
	inv_K,0
	inv_k,1
	inv_k,2
	inv_k,3

	color 3side x 4 scale = 12
	aug_color 3side x 4 scale = 12
	depth_gt

outputs #34 dict
	cam2cam 2side
	depth 4 scale
	sample 2side x 4 scale
	disp 4 scale
	color 2x4 #sides pred
	color_identity 2 x4
```
	
	
	


### framework modes
 - shared
 - ind
 - spv
 - rebuild

### model mode
 __Encoder(DepthEncoder)__ 
 model_mode[0]
 
- 1in
- 3in
- 3din

__DepthDecoder__
 model_mode[1]

- None

__PoseEncoder__[2]

- 3din
- c3d

__PoseDecoder__
 model_mode[3]

- fin-2out 
- fin-1out



	
	


### evaluation pipline



```apex
evaluate_depth.py

#step 0
	output = depth_decoder(encoder(input_color))
#step 1
	pred_disp, pred_depth_tmp = disp_to_depth(output[("disp", 0)], opt.min_depth, opt.max_depth)
#step 2

 	pred_disp = cv2.resize(pred_disp, (gt_width, gt_height))#1271x341 t0 128x640
    pred_depth = 1 / pred_disp# 也可以根据上面直接得到

#step 3
	pred_depth = pred_depth[mask]#并reshape成1d
	gt_depth = gt_depth[mask]

#step 4
	 if opt.median_scaling:
		ratio = np.median(gt_depth) / np.median(pred_depth)#中位数， 在eval的时候， 将pred值线性变化，尽量能使与gt接近即可
		ratios.append(ratio)
		pred_depth *= ratio

#step 5
 	pred_depth[pred_depth < MIN_DEPTH] = MIN_DEPTH#所有历史数据中最小的depth, 更新,
    pred_depth[pred_depth > MAX_DEPTH] = MAX_DEPTH#...
#step 6
    metric = compute_errors(gt_depth, pred_depth)
    
    
    

trainer.py
#step 0 batch_process
	disp = self.models["depth"](*features)
	outputs[("disp", 0, 0)] = disp[0]
	outputs[("disp", 0, 1)] = disp[1]
	outputs[("disp", 0, 2)] = disp[2]
	outputs[("disp", 0, 3)] = disp[3]

#step 1	generate_img_pred
	disp = outputs[("disp", 0, scale)]
 	disp = F.interpolate(disp, [height, width], mode="bilinear", align_corners=False)

	_, depth = disp_to_depth(disp)
    #depth = disp2depth(disp)
    #outputs[("depth", 0, scale)] = depth


# compute_metrics
 depth_pred_full = F.interpolate(outputs[("depth", 0, 0)], [full_height, full_width], mode="bilinear", align_corners=False)
        depth_pred = torch.clamp(
              depth_pred_full,
              min=min_depth,
              max=max_depth
          )

mask = depth_gt > depth_gt.min()
mask*=depth_gt< depth_gt.max()


depth_gt = depth_gt[mask]
depth_pred = depth_pred[mask]


depth_pred *= torch.median(depth_gt) / torch.median(depth_pred)

depth_pred = torch.clamp(depth_pred, min=min_depth, max=max_depth)#截断
depth_gt = torch.clamp(depth_gt,min=min_depth,max = max_depth)

metrics = compute_depth_errors(depth_gt, depth_pred)
    

```