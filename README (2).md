
# README - 3D Controller 
---
In this CPP project, we develop a controller for a drone in a 3D environment. The task is divided into five CPP functions. At the end, the desired individual thrusts of the four propellers are provided to the drone. 

We will then perform several flight scenarios and will observe how our controller performs and if the drone follows the desired path. 

### Controller Block Diagram (* borrowed from Udacity Self-Flying Lecture Video Slide)
<br>
<div class="container" style="width: 100%;">
 <div class="D1 col-sm-6" style="width: 80%;">
   <img src="3D_PID.png" height="100">
 </div>
</div>

### Drone dynamics
The drone can move along the three position axis $x$, $y$ and $z$. We choose the $z$ axis to be directed downward. The controller will track the drone's change in attitude in two coordinate systems. One in an inertial frame relative to the surroundings and the second in the body frame attached to the drone itself. 

## 1. GenerateMotorCommand() routine

<br>
<div class="container" style="width: 100%;">
 <div class="D1 col-sm-6" style="width: 40%;">
   <img src="Drone1.png" height="5">
 </div>
</div>
<br>
We will define the forces generated by the propellers 
$F_1 = k_f\omega^2_1$, $F_2 = k_f\omega^2_2$, $F_3 = k_f\omega^2_3$, $F_4 = k_f\omega^2_4$. The collective force directed upward $F_{total} = F_1 + F_2 + F_3 + F_4$.

The moment generated by the propeller is directed opposite of its rotation and is proportional to the square of the angular velocities.  

$$
\begin{align}
\tau_x &= (F_1 - F_2 + F_3 - F_4)l \\
\tau_y &= (F_1 + F_2 - F_3 - F_4)l \\
\tau_z &= -\tau_1 + \tau_2 + \tau_3 - \tau_4
\end{align}
$$

Where $\tau_1 = (k_m/k_f) \omega^2_1$, $\tau_2 = (k_m/k_f) \omega^2_2$, $\tau_3 = (k_m/k_f) \omega^2_3$, $\tau_4 = (k_m/k_f) \omega^2_4$. In our notation, the propellers 1 and 4 rotate in clockwise thus producing the moment in the counterclockwise direction with positive sign and propellers 2 and 3 rotate in counterclockwise thus the resulting moments are in opposite and have the negative signs. The final desired thrust can be calculated and delivered to the drone in the following GenerateMotorCommands() routine. 

As shown in the code below, the thrust and moments are converted to the appropriate 4 different desired thrust forces for the moments. We ensure that the dimensions of the drone are properly accounted for when calculating thrust from moments.

VehicleCommand QuadControl::GenerateMotorCommands(float collThrustCmd, V3F momentCmd)
{

    float l = L/1.4142135623730951;
    float f1 = collThrustCmd;
    float f2 = momentCmd.x/l;
    float f3 = momentCmd.y/l;
    float f4 = momentCmd.z/kappa;
  
    cmd.desiredThrustsN[0] = 0.25 * ( f1 + f2 + f3 - f4 ); // front left
    cmd.desiredThrustsN[1] = 0.25 * ( f1 - f2 + f3 + f4 ); // front right
    cmd.desiredThrustsN[2] = 0.25 * ( f1 + f2 - f3 + f4 ); // rear left
    cmd.desiredThrustsN[3] = -0.25 * (-f1 + f2 + f3 + f4 ); // rear right

  return cmd;
}



### 2.  Lateral controller 
The lateral controller will use a 2-cascaded P controller to command target values for elements of the drone's rotation matrix. The drone generates lateral acceleration by changing the body orientation which results in non-zero thrust in the desired direction. This will translate into the commanded acceleration along the axis $\ddot{x}_{\text{command}}$ and $\ddot{y}_{\text{command}}$. The control equations have the following form:

$$
\begin{align}
\ddot{x}_{\text{command}} &=  k_d^\dot{x}(k^x_p(x_t-x_a) + \dot{x}_t - \dot{x}_a)+ \ddot{x}_t \\
\end{align}
$$

for the $y$ direction the control equations will have the same form as above.

<br>
*The controller uses the local NE position and velocity to generate a commanded local acceleration.*

<br>
// returns a desired acceleration in global frame
<br>
V3F QuadControl::LateralPositionControl(V3F posCmd, V3F velCmd, V3F pos, V3F vel, V3F accelCmdFF)
{

    // make sure we don't have any incoming z-component
    accelCmdFF.z = 0;
    velCmd.z = 0;
    posCmd.z = pos.z;
    V3F velocity_cmd;
    V3F accelCmd;

    velocity_cmd = kpPosXY * ( posCmd - pos ) + velCmd;
    float velocity_norm = sqrt( velocity_cmd.x*velocity_cmd.x + velocity_cmd.y*velocity_cmd.y );
    if ( velocity_norm > maxSpeedXY ) {  velocity_cmd = velocity_cmd*maxSpeedXY/velocity_norm; }
    accelCmd = accelCmdFF + kpVelXY * ( velocity_cmd - vel );
    accelCmd.z = 0;

    return accelCmd;
}
<br>

### 3.  Roll-Pitch controller

The roll-pitch controller is a P controller responsible for commanding the roll and pitch rates ($p_c$ and $q_c$) in the body frame.  First, it sets the desired rate of change of the given matrix elements using a P controller. The collected thrust will be coverted to acceleration first.

*As shown in the code below, the controller uses the acceleration and thrust commands, in addition to the vehicle attitude to output a body rate command. The controller accounts for the non-linear transformation from local accelerations to body rates. Note that the drone's mass is accounted for when calculating the target angles.*


// returns a desired roll and pitch rate 
<br>
V3F QuadControl::RollPitchControl(V3F accelCmd, Quaternion<float> attitude, float collThrustCmd)
{

    V3F pqrCmd;
    Mat3x3F R = attitude.RotationMatrix_IwrtB();
    float target_R13, target_R23, p_cmd, q_cmd;
    ////////////////////////////// BEGIN STUDENT CODE ///////////////////////////
    float c_d = collThrustCmd/mass;
 
    if ( collThrustCmd > 0.0 ) {
        target_R13 = -CONSTRAIN(accelCmd.x/c_d, -maxTiltAngle, maxTiltAngle); //#-min(max(accelCmd.x/c_d, -maxTiltAngle), maxTiltAngle)
        target_R23 = -CONSTRAIN(accelCmd.y/c_d, -maxTiltAngle, maxTiltAngle); //#-min(max(accelCmd.y/c_d, -maxTiltAngle), maxTiltAngle)
      
        p_cmd = (1/R(2, 2)) *
              (-R(1, 0) * kpBank * (R(0, 2)-target_R13) + 
               R(0, 0) * kpBank * (R(1, 2)-target_R23));
        q_cmd = (1/R(2, 2)) * 
              (-R(1, 1) * kpBank * (R(0, 2)-target_R13) + 
               R(0, 1) * kpBank * (R(1, 2)-target_R23));
    }
    else { //:  # Otherwise command no rate
        //print("negative thrust command")
        p_cmd = 0.0;
        q_cmd = 0.0;
    }
  
    pqrCmd.x = p_cmd;
    pqrCmd.y = q_cmd;
    pqrCmd.z = 0.0;
    /////////////////////////////// END STUDENT CODE ////////////////////////////

    return pqrCmd;
}



### 4.  Body rate controller
The commanded roll, pitch, and yaw are collected by the body rate controller, and they are translated into the desired moments of each axis  in the body frame, which is the multiplication of the moment of inertia, $I_{xx}, I_{yy}, I_{zz}$ with the rotational accelerations along the axis in the body frame. 

$p_{\text{error}} = p_c - p$

$\tau_p= I_{xx} * k_{p-p} p_{\text{error}}$

$q_{\text{error}} = q_c - q$

$\tau_q= I_{yy} * k_{p-q} q_{\text{error}}$

$r_{\text{error}} = r_c - r$

$\tau_r= I_{zz} *k_{p-r} r_{\text{error}}$

<br>
*As shown in the CPP code below, the body rate controller is a proportional controller on body rates to commanded moments. The controller takes into account the moments of inertia, $Ixx, Iyy, Izz$, of the drone when calculating the commanded moments.*

V3F QuadControl::BodyRateControl(V3F pqrCmd, V3F pqr)
{
    V3F momentCmd;

    ////////////////////////////// BEGIN STUDENT CODE ///////////////////////////
    momentCmd.x = Ixx * kpPQR.x * ( pqrCmd.x - pqr.x ); 
    momentCmd.y = Iyy * kpPQR.y * ( pqrCmd.y - pqr.y ); 
    momentCmd.z = Izz * kpPQR.z * ( pqrCmd.z - pqr.z ); 
    /////////////////////////////// END STUDENT CODE ////////////////////////////

    return momentCmd;
}


### 5. Yaw controller

Control over yaw is decoupled from the other directions. A P controller is used to control the drone's yaw. It is desired to unwrap the radian angle yaw to a range of $[-\pi, \pi]$.

$r_c = k_p (\psi_t - \psi_a)$

*The yaw controller is a linear/proportional heading controller to yaw rate commands (non-linear transformation not required), as shown below.*

float QuadControl::YawControl(float yawCmd, float yaw)
{

    float yawRateCmd=0;
    ////////////////////////////// BEGIN STUDENT CODE ///////////////////////////
    float pi=3.14159265;
    float yaw_err = yawCmd - yaw;
    yaw_err = fmodf( yaw_err, pi );
    yawRateCmd = kpYaw * yaw_err;
    /////////////////////////////// END STUDENT CODE ////////////////////////////

    return yawRateCmd;

}


### 6.  Altitude Controller

Linear acceleration can be expressed by the next linear equation
$$
\begin{pmatrix} \ddot{x} \\ \ddot{y} \\ \ddot{z}\end{pmatrix}  = \begin{pmatrix} 0 \\ 0 \\ g\end{pmatrix} + R \begin{pmatrix} 0 \\ 0 \\ c \end{pmatrix} 
$$

where $R = R(\psi) \times R(\theta) \times R(\phi)$. The individual linear acceleration has the form of 

$$
\begin{align}
\ddot{x} &= c b^x \\ 
\ddot{y} &= c b^y \\ 
\ddot{z} &= c b^z +g
\end{align}
$$ 
where $b^x = R_{13}$, $b^y= R_{23}$ and $b^z = R_{33}$ are the elements of the last column of the rotation matrix. 

We are controlling the vertical acceleration: 

$$\bar{u}_1 = \ddot{z} = c b^z +g$$ 

Therefore 

$$c = -mass * (\bar{u}_1-g)/b^z$$  _(adjusted for mass and upward negative direction)_


In this exercise a 2-cascaded P controller + I controller is used for the altitude which results in: 

$$\bar{u}_1 =  k_{p-\dot{z}}(k_{p-z}(z_{t} - z_{a}) - \dot{z}_{a}) + \ddot{z}_t + k_{i-z} * integratedAltitudeError$$

<br>
*As shown in the code below, the controller uses both the down position and the down velocity to command thrust. We ensure that the output value is indeed thrust (the drone's mass needs to be accounted for) and that the thrust includes the non-linear effects from non-zero roll/pitch angles.*

*Additionally, the C++ altitude controller contains an integrator to handle the weight non-idealities presented in scenario 4.*

float QuadControl::AltitudeControl(float posZCmd, float velZCmd, float posZ, float velZ, Quaternion<float> attitude, float accelZCmd, float dt)
{
    Mat3x3F R = attitude.RotationMatrix_IwrtB();
    float thrust, hdot_cmd, acceleration_cmd, tmp_thrust;

    ////////////////////////////// BEGIN STUDENT CODE ///////////////////////////
    hdot_cmd = kpPosZ * (posZCmd - posZ) + velZCmd;
    hdot_cmd = CONSTRAIN(hdot_cmd, -maxDescentRate, maxAscentRate);
    integratedAltitudeError += (posZCmd - posZ) * dt;
    acceleration_cmd = accelZCmd + kpVelZ*(hdot_cmd - velZ) + KiPosZ*integratedAltitudeError;
    thrust = -mass * ( acceleration_cmd - CONST_GRAVITY ) / R(2,2);
    /////////////////////////////// END STUDENT CODE ////////////////////////////
  
    return thrust;
}


# 7. Tuning parameters to satisfy the project requirement

In order to test the developed drone controller, we executed the provided scenarios and able to pass the tests. 

## 7.1  Scenario 1 - Introduction

We balanced the drone weight to 0.5kg in order to keep the drone stable. The test passed.

## 7.2 Scenario 2 - Altitude control

In this scenario, we completed the GenerateMotorCommands() properly as explained in Section 1 above. To test the BodyRateControl(), we tuned the $kpPQR$ parameter to get the vehicle to stop spinning too much while preventing overshooting. For the RollPitchControl() test, we tuned $kpBank$ to minimize settling time and avoid overshoot. 

## 7.3 Scenario 3 - Position Control

We tuned the $kpPosXY, kpPosZ, kpVelXY, kpVelZ$ for the LateralPositionControl, and $kpYaw$ and the third z compomnet of the $kpPQR$ for the YawControl() test.

## 7.4 Scenario 4 - Non-idealities

Since the red drone is heavier than the other drones, we needed an integrated controller to mitigate the bias error. So that was the reason we implement the I controller along with the 2 cascaded P controller for the LateralControl().

## 7.5  Scenario 5 - Trajectory Follows

In this scenario, we followed the number 8 trajectory, and we were able to observe that adding the I term indeed helped the drone to follow the trajectory well. The test passed.

<br>
<div class="container" style="width: 100%;">
 <div class="D1 col-sm-6" style="width: 100%;">
   <img src="Scenario_5.png" height="50">
 </div>
</div>
<br>


