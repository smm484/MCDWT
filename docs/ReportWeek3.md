# (Week 3) Adaptative motion compensation based on the distortion of the prediction error

The objective of this issue has been to modify the MCDWT.py code to change the prediction based on the average [(BHA+BHC)/2] by the calculation based on the distortion of the prediction error [(BHA·ELC + BHC·ELA)/(ELA + ELC)], según [this](https://sistemas-multimedia.github.io/MCDWT/#x1-160006).

## Our experiments

1. In our first tests, we modified the code to calculate the BLA and BLC bands and implement the new formula:

	Code extracted from MCDWT.py
...

		AL = self.dwt.backward((aL, self.zero_H))
        BL = self.dwt.backward((bL, self.zero_H))
        CL = self.dwt.backward((cL, self.zero_H))
        AH = self.dwt.backward((self.zero_L, aH))
        BH = self.dwt.backward((self.zero_L, bH))
        CH = self.dwt.backward((self.zero_L, cH))
       
        BHA = generate_prediction(AL, BL, AH)
        BHC = generate_prediction(CL, BL, CH)
        '''
        Nueva Predicción BH
        '''
        BLA = generate_prediction(AL, BL, AL)
        BLC = generate_prediction(CL, BL, CL)
   

        ELA = BL - BLA
        ELC = BL - BLC
        if ((ELA + ELC).all == 0):
            prediction_BH = BHA
        else:
            prediction_BH = (BHA*ELC + BHC*ELA)/(ELA + ELC)

        '''
        Fin Nueva Predicción BH
        '''
        residue_BH = BH - prediction_BH
        residue_bH = self.dwt.forward(residue_BH)
        return residue_bH[1]
...

	
	In an attempt to control the division by 0, we use an if. But this did not solve the possible division by zero since the sum of the error could not be zero but could be zero at one point. 

2. After that, we focused on optimizing the calculation of the bands so as not to do any more calculations, so to optimize it we change the function generate_prediction by the following:


  		'''	
        Cálculo BHA, BLA, BHC, BLC optimizado
        '''

        flow = motion_estimation(AL, BL)
        BHA = estimate_frame(AH, flow)
        BLA = estimate_frame(AL, flow)
        flow = motion_estimation(CL, BL)
        BHC = estimate_frame(CH, flow)
        BLC = estimate_frame(CL, flow)

3. When updating the repository and see the last modification to correct the division by 0, we modify the code in the following way:

		ELA = BL - BLA
        ELC = BL - BLC
        '''Modificación para intentar corregir la división por 0'''
        SLA = 1 / (1+abs(ELA))
        SLC = 1 / (1+abs(ELC))
        prediction_BH = (BHA*SLA + BHC*SLC)/(SLA + SLC)

4. Finally, we introduce a parameter to choose one or another prediction (0-average or 1-distortion):

		'''
        Cálculo BHA, BLA, BHC, BLC optimizado
        '''
        flow = motion_estimation(AL, BL)
        BHA = estimate_frame(AH, flow)
        BLA = estimate_frame(AL, flow)
        flow = motion_estimation(CL, BL)
        BHC = estimate_frame(CH, flow)
        BLC = estimate_frame(CL, flow)

        if args.predictionerror == 1: 
            ELA = BL - BLA
            ELC = BL - BLC
            '''Modificación corregir la división por 0'''
            SLA = 1 / (1+abs(ELA))
            SLC = 1 / (1+abs(ELC))
         
            prediction_BH = (BHA*SLA + BHC*SLC)/(SLA + SLC)

            '''
            Fin Nueva Predicción BH
            '''
        else:
            prediction_BH = (BHA + BHC) / 2
        BH = residue_BH + prediction_BH
        bH = self.dwt.forward(BH)
        return bH[1]
        
        ...
        
        parser.add_argument("-e", "--predictionerror", default=1, type=int)  

    	args = parser.parse_args()

5. We modify the code to obtain a cleaner code

**prediction_0.py**

	def calcula_prediction():
    	prediction_BH = (BHA + BHC) / 2
    	return prediction_BH

**prediction_1.py**

	from MC.optical.motion import motion_estimation
	from MC.optical.motion import estimate_frame
	
	def calcula_prediction():
    	ELA = BL - BLA
    	ELC = BL - BLC
    	'''Modificación para corregir la división por 0'''
    	SLA = 1 / (1+abs(ELA))
    	SLC = 1 / (1+abs(ELC))
         
    	prediction_BH = (BHA*SLA + BHC*SLC)/(SLA + SLC)
    	return prediction_BH
    	
**MCDWT_AdaptCompensation.py**
	
	...
	
	'''New prediction'''
	prediction_BH = defprediction.calcula_prediction()
	
	...
	
	'''import according to parameter'''
	parser.add_argument("-e", "--predictionerror", default=0, type=int)  
	
	...
	
    if args.predictionerror == 0:
        import prediction_0 as defprediction
        print("Imported prediction_0")
    elif args.predictionerror == 1:
        import prediction_1 as defprediction
        print("Imported prediction_1")
 
        
## Our test

### New parameter to use:
**-e 0** -->	to make the prediction by the average

**-e 1** -->	to make the prediction by compensation
### Example:
	python3 -O ./MDWT.py -i ../sequences/stockholm/ -d /tmp/stockholm_
	MCDWT_AdaptCompensation.py -d /tmp/stockholm_ -m /tmp/mc_stockholm_ -e 0 
	MCDWT_AdaptCompensation.py -d /tmp/stockholm_ -m /tmp/mc_stockholm_ -e 1


 