# Gesture-recognition-system

import cv2
import numpy as np

# Initialize video capture
cap = cv2.VideoCapture(0)

# Background subtraction
bg_subtractor = cv2.createBackgroundSubtractorMOG2(history=500, varThreshold=25, detectShadows=False)

while True:
    ret, frame = cap.read()
    if not ret:
        break
    
    # Flip the frame horizontally
    frame = cv2.flip(frame, 1)
    
    # Define region of interest (ROI)
    roi = frame[100:400, 100:400]
    cv2.rectangle(frame, (100, 100), (400, 400), (0, 255, 0), 2)
    
    # Apply background subtraction
    fg_mask = bg_subtractor.apply(roi)
    _, fg_mask = cv2.threshold(fg_mask, 200, 255, cv2.THRESH_BINARY)
    
    # Apply skin color detection in YCrCb space (better for skin detection)
    ycrcb = cv2.cvtColor(roi, cv2.COLOR_BGR2YCrCb)
    lower_skin = np.array([0, 135, 85], dtype=np.uint8)
    upper_skin = np.array([255, 180, 135], dtype=np.uint8)
    skin_mask = cv2.inRange(ycrcb, lower_skin, upper_skin)
    
    # Combine masks
    final_mask = cv2.bitwise_and(fg_mask, skin_mask)
    
    # Apply morphological operations
    kernel = np.ones((3, 3), np.uint8)
    final_mask = cv2.erode(final_mask, kernel, iterations=1)
    final_mask = cv2.dilate(final_mask, kernel, iterations=2)
    final_mask = cv2.GaussianBlur(final_mask, (5, 5), 0)
    
    # Find contours
    contours, _ = cv2.findContours(final_mask, cv2.RETR_TREE, cv2.CHAIN_APPROX_SIMPLE)
    
    if len(contours) > 0:
        # Find the largest contour
        max_contour = max(contours, key=lambda x: cv2.contourArea(x))
        
        # Create convex hull
        hull = cv2.convexHull(max_contour, returnPoints=False)
        
        # Find convexity defects
        if len(hull) > 3:
            defects = cv2.convexityDefects(max_contour, hull)
            
            if defects is not None:
                finger_count = 0
                defect_points = []
                
                for i in range(defects.shape[0]):
                    s, e, f, d = defects[i, 0]
                    start = tuple(max_contour[s][0])
                    end = tuple(max_contour[e][0])
                    far = tuple(max_contour[f][0])
                    
                    # Calculate triangle sides
                    a = np.sqrt((end[0] - start[0])**2 + (end[1] - start[1])**2)
                    b = np.sqrt((far[0] - start[0])**2 + (far[1] - start[1])**2)
                    c = np.sqrt((end[0] - far[0])**2 + (end[1] - far[1])**2)
                    
                    # Calculate angle
                    angle = np.arccos((b**2 + c**2 - a**2)/(2*b*c + 1e-10)) * 180/np.pi
                    
                    # Ignore angles > 90 degrees and points too close
                    if angle < 90 and d > 10000:
                        finger_count += 1
                        defect_points.append(far)
                
                # Draw defects and count fingers
                for point in defect_points:
                    cv2.circle(roi, point, 8, [0, 255, 0], -1)
                
                # Adjust finger count (1 defect = 2 fingers, etc.)
                finger_count = min(finger_count + 1, 5)
                
                # Display finger count
                cv2.putText(frame, f"Fingers: {finger_count}", (50, 50), 
                          cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 2)
                
                # Draw contour
                cv2.drawContours(roi, [max_contour], -1, (255, 0, 0), 2)
    
    # Show the frames
    cv2.imshow('Mask', final_mask)
    cv2.imshow('Frame', frame)
    
    # Exit on 'q' key press
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

# Release resources
cap.release()
cv2.destroyAllWindows()
