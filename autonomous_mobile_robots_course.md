# Autonomous Mobile Robots Course

****

## Introduction

### Segment 0
- see-think-act cycle
    - motion control
        - rolling and sliding constraints
        - motion function
    - perception
        - laser scanners (time of flight), cameras (cheap, noisy)
        - information extraction
            - edge detection
            - keypoint features (invariant to rotation, scaling, etc; gradiant in different directions; FAST, SURF, SIFT, ...)
            - keypoint matching
    - localization
        - see -> act -> see -> belief update (information fusion)
    - cognition (where am IU? How do I get there?)
        - global path planning (graph search)
        - local path planning (local collision avoidance)