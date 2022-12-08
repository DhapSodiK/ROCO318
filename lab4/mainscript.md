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

function sysCall_init() 
    usensors={-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1}
    for i=1,16,1 do
        usensors[i]=sim.getObjectHandle("Pioneer_p3dx_ultrasonicSensor"..i)
    end
    motorLeft=sim.getObjectHandle("Pioneer_p3dx_leftMotor")
    motorRight=sim.getObjectHandle("Pioneer_p3dx_rightMotor")
    noDetectionDist=0.5
    maxDetectionDist=0.2
    detect={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0}
    braitenbergL={-0.2,-0.4,-0.6,-0.8,-1,-1.2,-1.4,-1.6, 0.0,0.0,0.0,0.0,0.0,0.0,0.0,0.0}
    braitenbergR={-1.6,-1.4,-1.2,-1,-0.8,-0.6,-0.4,-0.2, 0.0,0.0,0.0,0.0,0.0,0.0,0.0,0.0}
    v0=2
    
    base_link = sim.getObjectHandle('base_link')
    center_wheel = sim.getObjectHandle('center_wheel')
    front_sonar = sim.getObjectHandle('front_sonar')
    lms100 = sim.getObjectHandle('lms100')
    p3dx_left_wheel = sim.getObjectHandle('p3dx_left_wheel')
    p3dx_right_wheel = sim.getObjectHandle('p3dx_right_wheel')
    swivel = sim.getObjectHandle('swivel')
    top_plate = sim.getObjectHandle('top_plate')
    center_hubcap = sim.getObjectHandle('center_hubcap')
    p3dx_right_hubcap = sim.getObjectHandle('p3dx_right_hubcap')
    p3dx_left_hubcap = sim.getObjectHandle('p3dx_left_hubcap')
    
    odom = sim.getObjectHandle('odom')
    map = sim.getObjectHandle('map')
    fastHokuyo_ROS = sim.getObjectHandle('fastHokuyo_ROS')
    
    gyro_link = sim.getObjectHandle('gyro_link')
    Accelerometer = sim.getObjectHandle('Accelerometer')
    GyroSensor = sim.getObjectHandle('GyroSensor')
    
    camera_link = sim.getObjectHandle('camera_link')
    camera_depth_frame = sim.getObjectHandle('camera_depth_frame')
    camera_rgb_frame = sim.getObjectHandle('camera_rgb_frame')
    PointCloud_frame = sim.getObjectHandle('PointCloud_frame')
    
    odom_data_pub = simROS.advertise('/odom', 'nav_msgs/Odometry')
    subscriber_name = simROS.subscribe('/cmd_vel','geometry_msgs/Twist','wheels_cmd_vel_callback')
    
    
end


function wheels_cmd_vel_callback(msg)

   -- Your code here  

end

-- This is a very simple EXAMPLE navigation program, which avoids obstacles using the Braitenberg algorithm


function sysCall_cleanup() 
 
end 

function sysCall_actuation() 
    for i=1,16,1 do
        res,dist=sim.readProximitySensor(usensors[i])
        if (res>0) and (dist<noDetectionDist) then
            if (dist<maxDetectionDist) then
                dist=maxDetectionDist
            end
            detect[i]=1-((dist-maxDetectionDist)/(noDetectionDist-maxDetectionDist))
        else
            detect[i]=0
        end
    end
    
    vLeft=v0
    vRight=v0
    
    for i=1,16,1 do
        vLeft=vLeft+braitenbergL[i]*detect[i]
        vRight=vRight+braitenbergR[i]*detect[i]
    end
    
    sim.setJointTargetVelocity(motorLeft,vLeft)
    sim.setJointTargetVelocity(motorRight,vRight)
    
    simROS.publish(odom_data_pub, rosGetOdomData())
end 

rosGetOdomData=function()

	msg = 
        {
            header=
            {
                seq=0,
                stamp= simROS.getTime(),
                frame_id= 'odom'
            },
            child_frame_id= 'base_link',
            pose=   
            {
                pose=  
                {
                    position= 
                    {
                        x=getTransformStamped(base_link, 'base_link', odom,'odom').transform.translation.x,
                        y=getTransformStamped(base_link, 'base_link', odom,'odom').transform.translation.y,
                        z=getTransformStamped(base_link, 'base_link', odom,'odom').transform.translation.z
                    },
                    orientation= 
                    {
                        x=getTransformStamped(base_link, 'base_link', odom,'odom').transform.rotation.x,
                        y=getTransformStamped(base_link, 'base_link', odom,'odom').transform.rotation.y,
                        z=getTransformStamped(base_link, 'base_link', odom,'odom').transform.rotation.z,
                        w=getTransformStamped(base_link, 'base_link', odom,'odom').transform.rotation.w
                    }
                }
            },
            twist=
            {
                twist=
                {
                    linear=
                    {
                        x=sim.getObjectVelocity(base_link)[1],
                        y=sim.getObjectVelocity(base_link)[2],
                        z=sim.getObjectVelocity(base_link)[3]
                    },
                    angular=
                    {
                        x=sim.getObjectVelocity(base_link)[1],
                        y=sim.getObjectVelocity(base_link)[2],
                        z=sim.getObjectVelocity(base_link)[3]
                    }
                }
            }
        }

	return msg
end

function sysCall_sensing()
    transforms = {
        getTransformStamped(lms100, 'lms100', base_link,'base_link'),
        getTransformStamped(center_wheel, 'center_wheel', base_link,'base_link'),
        getTransformStamped(front_sonar, 'front_sonar', base_link,'base_link'),
        getTransformStamped(p3dx_left_wheel, 'p3dx_left_wheel', base_link,'base_link'),
        getTransformStamped(p3dx_right_wheel, 'p3dx_right_wheel', base_link,'base_link'),
        getTransformStamped(swivel, 'swivel', base_link,'base_link'),
        getTransformStamped(top_plate, 'top_plate', base_link,'base_link'),
        getTransformStamped(center_hubcap, 'center_hubcap', center_wheel,'center_wheel'),
        getTransformStamped(p3dx_right_hubcap, 'p3dx_right_hubcap', p3dx_right_wheel,'p3dx_right_wheel'),
        getTransformStamped(p3dx_left_hubcap, 'p3dx_left_hubcap', p3dx_left_wheel,'p3dx_left_wheel'),
        
        getTransformStamped(base_link, 'base_link', odom,'odom'),
        --getTransformStamped(odom, 'odom', map,'map'),
        getTransformStamped(fastHokuyo_ROS, 'fastHokuyo_ROS', base_link, 'base_link'),
        
        getTransformStamped(gyro_link, 'gyro_link', base_link,'base_link'),
        getTransformStamped(Accelerometer, 'Accelerometer', gyro_link, 'gyro_link'),
        getTransformStamped(GyroSensor, 'GyroSensor', gyro_link, 'gyro_link'),
        
        --getTransformStamped(camera_link, 'camera_link', base_link,'base_link'),
        --getTransformStamped(camera_depth_frame, 'camera_depth_frame', camera_link,'camera_link'),
        --getTransformStamped(camera_rgb_frame, 'camera_rgb_frame', camera_link,'camera_link'),
        getTransformStamped(PointCloud_frame, 'PointCloud_frame', camera_link, 'camera_link'),
        
        getTransformStamped(camera_link, 'camera_link', map, 'map'),

        
    }
    simROS.sendTransforms(transforms)
end
