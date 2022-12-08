function sysCall_init() 
    depthCam=sim.getObject('./depth')
    depthView=sim.floatingViewAdd(0.9,0.9,0.2,0.2,0)
    sim.adjustView(depthView,depthCam,64)

    colorCam=sim.getObject('./rgb')
    colorView=sim.floatingViewAdd(0.69,0.9,0.2,0.2,0)
    sim.adjustView(colorView,colorCam,64)
    
    --pointCloud=sim.getObject('./PointCloud')
    --sim.adjustView(pointCloud,pointCloud,64)
    
    rgb_info_pub = simROS.advertise('/camera/camera_info', 'sensor_msgs/CameraInfo')
    simROS.publisherTreatUInt8ArrayAsString(rgb_info_pub)
    
    rgb_pub = simROS.advertise('/camera/rgb/image_raw', 'sensor_msgs/Image')
    simROS.publisherTreatUInt8ArrayAsString(rgb_pub)
    
    depth_image_pub=simROS.advertise('/camera/depth/image_raw', 'sensor_msgs/Image')
    simROS.publisherTreatUInt8ArrayAsString(depth_image_pub)
    
    --cloud_out = simROS.advertise('/camera/pointCloud/image_raw', 'sensor_msgs/Image')
    --simROS.publisherTreatUInt8ArrayAsString(cloud_out)
    
    --depth_pub = simROS.advertise('/Float32MultiArray_in', 'std_msgs/Float32MultiArray')
    
    camResolution = sim.getVisionSensorResolution(colorCam)
    print('camResolution[1]: '..camResolution[1])
    print('camResolution[2]: '..camResolution[2])
    result, perspectivAngle = sim.getObjectFloatParameter(colorCam,sim.visionfloatparam_perspective_angle)
    print('perspectivAngle: '..perspectivAngle)
    f_x = (camResolution[1]/2.) / math.tan(perspectivAngle/2.)
    f_y = f_x
    print('f_x: '..f_x)
    print('f_y: '..f_y)
    result, nearClippingPlane = sim.getObjectFloatParameter(colorCam,sim.visionfloatparam_near_clipping)
    nearClippingPlane=nearClippingPlane*1000
    print('nearClippingPlane: '..nearClippingPlane)
    result, farClippingPlane=sim.getObjectFloatParameter(colorCam,sim.visionfloatparam_far_clipping)
    farClippingPlane=farClippingPlane*1000
    print('farClippingPlane: '..farClippingPlane)
    
    map=sim.getObjectHandle('map')
    camera_link=sim.getObjectHandle('camera_link')
    camera_rgb_frame=sim.getObjectHandle('camera_rgb_frame')
    camera_depth_frame=sim.getObjectHandle('camera_depth_frame')
    
    depth_pub = simROS.advertise('/Float32MultiArray_in', 'std_msgs/Float32MultiArray')
    
    
end
function sysCall_sensing()  
    transforms = {
        getTransformStamped(camera_link, 'camera_link', map,'map'),
        getTransformStamped(camera_depth_frame, 'camera_depth_frame', camera_link,'camera_link'),
        getTransformStamped(camera_rgb_frame, 'camera_rgb_frame', camera_link,'camera_link'),
        }
        
        simROS.sendTransforms(transforms)
        
    t = simROS.getTime()
    
    tb_rgb_info = {}
    tb_rgb_info['header']={stamp=t, frame_id="camera_depth_frame"}
    tb_rgb_info['height']= camResolution[2]
    tb_rgb_info['width']= camResolution[1]
    tb_rgb_info['distortion_model']='plumb_bob'
    tb_rgb_info['D'] = {0, 0, 0, 0, 0}
    tb_rgb_info['K'] = {f_x, 0, camResolution[1]/2, 0, f_y, camResolution[2]/2, 0, 0, 1}
    tb_rgb_info['R'] = {1, 0, 0, 0, 1, 0, 0, 0, 1}
    tb_rgb_info['P'] = {f_x, 0, camResolution[1]/2, 0, 0, f_y, camResolution[2]/2, 0, 0, 0, 1, 0}
    tb_rgb_info['binning_x']= 0
    tb_rgb_info['binning_y']= 0
    simROS.publish(rgb_info_pub, tb_rgb_info)
    
    image=sim.getVisionSensorImage(colorCam,0,0,0,0,1)
    sim.transformImage(image,{camResolution[1],camResolution[2]},4) -- vertical flip
    tb_rgb={}
    tb_rgb['header']={stamp=t, frame_id="camera_depth_frame"}
    tb_rgb['height']= camResolution[2]
    tb_rgb['width'] = camResolution[1]
    tb_rgb['encoding']='rgb8'
    tb_rgb['step']= camResolution[1] * 3
    tb_rgb['data'] = image 
    simROS.publish(rgb_pub, tb_rgb)
    
    depth_image = sim.getVisionSensorDepthBuffer(depthCam+sim.handleflag_codedstring)
    depth_image=sim.transformBuffer(depth_image,sim.buffer_float,farClippingPlane-nearClippingPlane,nearClippingPlane,sim.buffer_uint16)
    image_depth={}
    image_depth['header']={seq=0,stamp = t, frame_id="camera_depth_frame"}
    image_depth['height']=camResolution[2]
    image_depth['width']=camResolution[1]
    image_depth['encoding']='16UC1' 
    image_depth['is_bigendian']=1
    image_depth['step']= camResolution[1] * 2 
    image_depth['data'] = depth_image
    simROS.publish(depth_image_pub,image_depth)
    
    --Float32MultiArray_in = sim.getVisionSensorDepthBuffer(depthCam+sim.handleflag_codedstring)
    --Float32MultiArray_in=sim.transformBuffer(Float32MultiArray_in,sim.buffer_float,farClippingPlane-nearClippingPlane,nearClippingPlane,sim.buffer_uint16)
    --pc_3d={}
    --pc_3d['header']={seq=0,stamp = t, frame_id="camera_depth_frame"}
    --pc_3d['height']=480
    --pc_3d['width']=640
    --pc_3d['encoding']='16UC1' 
    --pc_3d['step']= camResolution[1] * 3
    --pc_3d['data'] = pc_image
    --simROS.publish(depth_pub,pc_3d)

    tb_depth={}
    -- depthVisionSensor_m = depthVisionSensor +  sim.handleflag_depthbuffer --  if you wish to retrieve values in meters
    tb_depth['data']=sim.getVisionSensorDepthBuffer(depthCam)
    simROS.publish(depth_pub, tb_depth)
end

function getTransformStamped(objHandle,name,relTo,relToName)
    -- This function retrieves the stamped transform for a specific object
    t=simROS.getTime()
    p=sim.getObjectPosition(objHandle,relTo)
    o=sim.getObjectQuaternion(objHandle,relTo)
    return {
        header={
            stamp=t,
            frame_id=relToName
        },
        child_frame_id=name,
        transform={
            translation={x=p[1],y=p[2],z=p[3]},
            rotation={x=o[1],y=o[2],z=o[3],w=o[4]}
        }
    }
end

function sysCall_cleanup() 
 
end 


