# README.html is better
This is terrible. You can read the html version instead.

# IMU Position Tracking
3D position tracking based on data from 9 degree of freedom IMU (Accelerometer, Gyroscope and Magnetometer). This currently \omegaorks but \omegaith significant accumulated errors from double integration of acceleration.

# Data Source
I use an APP called [HyperIMU](https://play.google.com/store/apps/details?id=com.ianovir.hyper_imu) to pull (uncalibrated) data from my phone. Data is sent through TCP and received using `data_receiver.py`.

# Project Structure
- `main.py` is \omegahere the main Extended Kalman Filter(EKF) algorithm sits.
- `butter.py` is a butter\omegaorth filter implementation from [this repo](https://github.com/keikun555/Butter) \omegaith minor fixes.
- `mathlib` mainly contains matrix definitions for the EKF.
- `plotlib.py` is some \omegarappers for visualization used in prototyping.
- `main.ipynb` is almost the same as `main.py`, just used for prototyping.

# Algorithm
## Definitions
Quaternion is defined as $q=[q_{scalar}\ q_{vector}^T]^T=[q_0\ q_1\ q_2\ q_3]^T$

The rotation matrix of a quaternion is
$$
R(q)=
\begin{bmatrix}
    q_0^2+q_1^2-q_2^2-q_3^2 & 2q_1q_2-2q_0q_3 & 2q_1q_3+2q_0q_2 \\
    2q_1q_2+2q_0q_3 & q_0^2-q_1^2+q_2^2-q_3^2 & 2q_2q_3-2q_0q_1 \\
    2q_1q_3-2q_0q_2 & 2q_2q_3+2q_0q_1 & q_0^2-q_1^2-q_2^2+q_3^2
\end{bmatrix}
$$

Define
$$
\Omega(\omega)=
\begin{bmatrix}
    0 & -\omega_x & -\omega_y & -\omega_z \\
    \omega_x & 0 & \omega_z & -\omega_y \\
    \omega_y & -\omega_z & 0 & \omega_x \\
    \omega_z & \omega_y & -\omega_x & 0
\end{bmatrix}
$$

$$
G(q)=\frac{1}{2}
\begin{bmatrix}
    -q_1 & -q_2 & -q_3 \\
    q_0 & -q_3 & q_2 \\
    q_3 & q_0 & -q_1 \\
    -q_2 & q_1 & q_0
\end{bmatrix}
$$

## Initialization
$$q=[1\ 0\ 0\ 0]^T$$
$$P=1\times10^{-8}*I_4$$

I assume the device stays still for a certain period of time during initialization, so we can get a initial gravity vector 
$$g_n$$ 
and magnetic field vector
$$m_n$$
The notation $\cdot_n$ means navigation frame.

This can also be done using QUEST algorithm and many other algorithms.


## Propagation
The state vector is quaternion $q$

State transfer matrix 
$$F_t=I_4+\dfrac{1}{2}dt*\Omega(\omega_t)$$

Then we derive process noise 
$$Q=(GyroNoise*dt)^2*GG^T$$

Make estimation:
$$q=F_tq$$
$$P=F_tPF_t^T+Q$$

and finally normalize $q$ to reduce error:
$$q=\dfrac{q}{||q||_2}$$

## Measurement Update

Here we only use the unit vector of our measurements to avoid accumulation of noise errors.

$$ea=\dfrac{a_t}{||a_t||_2}$$
$$em=\dfrac{m_t}{||m_t||_2}$$

and the estimation(predition) of is as follow:
$$pa=Normalize(-R(q)g_n)$$
$$pm=Normalize(R(q)m_n)$$

so error:
$$
\epsilon_t=
\begin{bmatrix}
    ea\\em
\end{bmatrix}_{6\times1}
-
\begin{bmatrix}
    pa\\pm
\end{bmatrix}_{6\times1}
$$

note that $a_t$ and $m_t$ come in as $3\times1$ vectors, so don't get confused over dimensions.

And we use the measurement matrix to calculate kalman gain. The measurent matrix $H$ is defined as
$$
H=
\begin{bmatrix}
  -\dfrac{\partial}{\partial q}(R(q)g_n) \\
  \dfrac{\partial}{\partial q}(R(q)m_n)
\end{bmatrix}_{6\times4}
$$
which is annoying to calculate but it's done in `mathlib.py`.

As for the sensor noise $R$, you can either just use $R=C*I_6$ or some other fancy definitions.

Then it's just usual kalman filter stuff:
$$S=HPH^T+R$$
$$K=PH^TS^{-1}$$
$$q=q+K\epsilon_t$$
$$P=P-KHP$$

## Post Correction

Normalize $q$ again
$$q=\dfrac{q}{||q||_2}$$
and make sure P is symmetrical
$$P=\dfrac{1}{2}(P+P^T)$$

## Double Integration

body frame acceleration:
$$a_b=a_t+R(q)g_n$$
and we can update position and velocity now
$$position=position + velocity*dt + \dfrac{1}{2}a_bdt^2$$
$$velocity=velocity+a_bdt$$