# Design Document: RTI Kit for COPD

## Overview

The RTI Kit for COPD is a comprehensive wearable medical device system that provides continuous monitoring, predictive analytics, and personalized interventions for patients with Chronic Obstructive Pulmonary Disease. The system architecture leverages AWS cloud services to deliver real-time data processing, machine learning-based exacerbation prediction, and clinical decision support.

The system consists of:
- **Wearable Device**: Multi-sensor hardware for continuous vital sign monitoring
- **Patient Mobile App**: iOS/Android application for patient engagement and self-management
- **Clinical Dashboard**: Web-based interface for healthcare providers
- **Caregiver Portal**: Web/mobile interface for family caregivers
- **AWS Backend**: Cloud infrastructure for data processing, storage, and analytics
- **ML Pipeline**: Machine learning models for exacerbation prediction and risk scoring

The design prioritizes clinical accuracy, patient safety, regulatory compliance, and seamless integration with existing healthcare workflows.

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                         RTI Kit for COPD                             │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────┐         ┌──────────────────┐         ┌──────────────────┐
│  Wearable Device │────────▶│   AWS IoT Core   │────────▶│  Data Processing │
│  - SpO2 Sensor   │         │  - MQTT Broker   │         │  - Lambda        │
│  - HR Monitor    │         │  - Device Shadow │         │  - Kinesis       │
│  - Accelerometer │         │  - Rules Engine  │         │  - Step Functions│
│  - Microphone    │         └──────────────────┘         └──────────────────┘
└──────────────────┘                  │                             │
                                      │                             ▼
┌──────────────────┐         ┌──────────────────┐         ┌──────────────────┐
│   Patient App    │◀────────│   API Gateway    │◀────────│  ML Pipeline     │
│  - Symptom Log   │         │  - REST API      │         │  - SageMaker     │
│  - Medication    │         │  - WebSocket     │         │  - Prediction    │
│  - Exercises     │         │  - Auth (Cognito)│         │  - Risk Scoring  │
└──────────────────┘         └──────────────────┘         └──────────────────┘
                                      │                             │
┌──────────────────┐                 │                             ▼
│ Clinical         │◀────────────────┘                   ┌──────────────────┐
│ Dashboard        │                                     │  Data Storage    │
│  - Patient List  │                                     │  - DynamoDB      │
│  - Alerts        │                                     │  - S3            │
│  - Reports       │                                     │  - Timestream    │
└──────────────────┘                                     └──────────────────┘
                                                                  │
┌──────────────────┐                                             │
│ Caregiver Portal │◀────────────────────────────────────────────┘
│  - Monitoring    │
│  - Alerts        │
└──────────────────┘
```

### Data Flow

1. **Sensor Data Collection**: Wearable device collects vital signs (SpO2, HR, RR, activity) at 1Hz
2. **Local Processing**: Device performs initial filtering and anomaly detection
3. **Data Transmission**: Aggregated data sent to AWS IoT Core every 5 minutes via MQTT
4. **Ingestion**: IoT Rules Engine routes data to Kinesis Data Streams
5. **Processing**: Lambda functions process streams, calculate metrics, detect anomalies
6. **ML Inference**: SageMaker endpoint predicts exacerbation risk
7. **Storage**: Processed data stored in DynamoDB (recent) and S3 (historical)
8. **Notification**: Critical alerts trigger SNS notifications to patients, caregivers, providers
9. **Visualization**: Dashboards query data via API Gateway for real-time display


## Components and Interfaces

### 1. Wearable Device Hardware

**Sensors:**
- **Pulse Oximeter**: Measures SpO2 and heart rate (Maxim MAX30102 or equivalent)
- **Respiratory Sensor**: Measures breathing rate via chest impedance or accelerometer
- **3-Axis Accelerometer**: Tracks activity levels and body position
- **Microphone**: Captures cough sounds and lung sounds
- **Temperature Sensor**: Monitors body temperature
- **Ambient Sensors**: Temperature, humidity, pressure

**Processing:**
- **MCU**: ARM Cortex-M4 with DSP capabilities
- **Memory**: 512KB RAM, 2MB Flash
- **Connectivity**: BLE 5.0, Wi-Fi 802.11n, LTE Cat-M1
- **Battery**: 500mAh Li-Po, 3-5 day life
- **Charging**: USB-C, wireless Qi charging

**Firmware Interface:**
```python
class WearableDevice:
    def read_spo2() -> tuple[float, float]:
        """Returns (spo2_percentage, heart_rate_bpm)"""
        
    def read_respiratory_rate() -> float:
        """Returns breaths per minute"""
        
    def read_activity_level() -> ActivityLevel:
        """Returns activity classification (sedentary, light, moderate, vigorous)"""
        
    def detect_cough() -> CoughEvent:
        """Returns cough detection with intensity and timestamp"""
        
    def get_battery_level() -> float:
        """Returns battery percentage (0-100)"""
        
    def transmit_data(data: SensorData) -> bool:
        """Sends data to AWS IoT Core via MQTT"""
```

### 2. AWS IoT Core Integration

**Device Registry:**
- Each wearable device registered with unique Thing Name
- Device certificates for mutual TLS authentication
- Device Shadow for state synchronization

**MQTT Topics:**
```
copd/devices/{device_id}/vitals          # Vital signs data
copd/devices/{device_id}/events          # Discrete events (cough, medication)
copd/devices/{device_id}/status          # Device health and battery
copd/devices/{device_id}/commands        # Downlink commands
```

**IoT Rules:**
```sql
-- Route vital signs to Kinesis
SELECT * FROM 'copd/devices/+/vitals'

-- Trigger immediate alert on critical SpO2
SELECT * FROM 'copd/devices/+/vitals' 
WHERE spo2 < 88

-- Route events to DynamoDB
SELECT * FROM 'copd/devices/+/events'
```

### 3. Data Processing Pipeline

**Lambda Functions:**

```python
def process_vital_signs(event: KinesisEvent) -> ProcessedVitals:
    """
    Processes incoming vital signs data:
    - Validates data quality
    - Calculates rolling averages
    - Detects anomalies vs baseline
    - Triggers ML inference
    """
    
def calculate_risk_score(patient_id: str, vitals: VitalSigns, 
                         symptoms: Symptoms, environment: EnvironmentData) -> RiskScore:
    """
    Calculates composite risk score (0-100):
    - Vital signs deviation from baseline (40%)
    - Symptom severity (30%)
    - Medication adherence (15%)
    - Environmental factors (15%)
    """
    
def detect_exacerbation_pattern(patient_id: str, 
                                window_days: int = 7) -> ExacerbationRisk:
    """
    Analyzes trends over time window:
    - Increasing respiratory rate
    - Declining SpO2
    - Increased rescue inhaler use
    - Worsening symptoms
    """
```

**Step Functions Workflow:**
```
Start → Validate Data → Calculate Metrics → ML Inference → Risk Scoring 
  → Alert Decision → Notification → Store Results → End
```

### 4. Machine Learning Pipeline

**Exacerbation Prediction Model:**

**Input Features (n=25):**
- Vital signs: SpO2 (mean, std, min), HR (mean, std), RR (mean, std)
- Activity: Daily steps, exercise minutes, sleep quality
- Symptoms: CAT score, mMRC scale, cough frequency
- Medication: Adherence %, rescue inhaler count
- Environment: AQI, temperature, humidity
- Historical: Days since last exacerbation, exacerbation count (90 days)
- Spirometry: FEV1 % predicted, FEV1/FVC ratio

**Model Architecture:**
- Algorithm: Gradient Boosted Trees (XGBoost)
- Training: Supervised learning on labeled exacerbation events
- Output: Probability of exacerbation in next 3-7 days
- Threshold: >0.7 probability = high risk alert

**Training Pipeline:**
```python
def train_exacerbation_model(training_data: DataFrame) -> Model:
    """
    Trains XGBoost model:
    - Feature engineering and normalization
    - Handle class imbalance (SMOTE)
    - Cross-validation (5-fold)
    - Hyperparameter tuning
    - Model evaluation (AUC-ROC, sensitivity, specificity)
    """
    
def personalize_model(patient_id: str, base_model: Model) -> Model:
    """
    Fine-tunes model for individual patient:
    - Uses patient's 14-day baseline period
    - Adjusts thresholds based on patient history
    - Updates monthly with new data
    """
```

**SageMaker Deployment:**
- Real-time endpoint for immediate predictions
- Batch transform for daily risk scoring
- Model monitoring for drift detection
- A/B testing for model updates

### 5. Patient Mobile Application

**Technology Stack:**
- React Native for cross-platform (iOS/Android)
- Redux for state management
- AWS Amplify for backend integration
- HealthKit/Google Fit integration

**Key Screens:**

```typescript
interface PatientApp {
  // Dashboard
  DashboardScreen: {
    currentVitals: VitalSigns;
    riskScore: number;
    todaysSummary: DailySummary;
    alerts: Alert[];
  };
  
  // Symptom Logging
  SymptomScreen: {
    catAssessment: CATScore;
    symptomDiary: SymptomEntry[];
    voiceRecording: () => void;  // AWS HealthScribe
  };
  
  // Medication Tracking
  MedicationScreen: {
    scheduledMeds: Medication[];
    adherenceRate: number;
    inhalerLog: InhalerEvent[];
  };
  
  // Pulmonary Rehabilitation
  ExerciseScreen: {
    dailyExercises: Exercise[];
    breathingTechniques: BreathingGuide[];
    progressTracking: ExerciseProgress;
  };
  
  // Action Plan
  ActionPlanScreen: {
    currentZone: 'green' | 'yellow' | 'red';
    actionSteps: ActionStep[];
    emergencyContacts: Contact[];
  };
}
```

### 6. Clinical Dashboard

**Technology Stack:**
- React for web frontend
- Material-UI for components
- D3.js for data visualization
- WebSocket for real-time updates

**Key Features:**

```typescript
interface ClinicalDashboard {
  // Patient List
  PatientListView: {
    patients: Patient[];
    filterByRisk: (level: RiskLevel) => Patient[];
    sortByPriority: () => Patient[];
  };
  
  // Patient Detail
  PatientDetailView: {
    demographics: PatientInfo;
    currentVitals: VitalSigns;
    vitalsTrends: TimeSeriesChart;
    riskScore: RiskScoreDisplay;
    alerts: Alert[];
    medications: MedicationList;
    spirometry: SpirometryData;
    goldStage: GOLDClassification;
  };
  
  // Analytics
  AnalyticsView: {
    populationMetrics: PopulationStats;
    exacerbationRates: ExacerbationAnalytics;
    outcomesTrends: OutcomesChart;
  };
  
  // Telemedicine
  TelemedicineView: {
    videoConsult: VideoSession;
    patientData: IntegratedData;
    clinicalNotes: HealthScribeNotes;
  };
}
```


### 7. AWS HealthScribe Integration

**Voice-Based Symptom Reporting:**

```python
class HealthScribeIntegration:
    def record_symptom_report(audio: AudioStream) -> SymptomReport:
        """
        Processes voice recording:
        1. Upload audio to S3
        2. Invoke HealthScribe API
        3. Extract structured data
        4. Generate clinical note
        """
        
    def extract_copd_symptoms(transcript: str) -> COPDSymptoms:
        """
        Extracts COPD-specific information:
        - Dyspnea severity (mMRC scale)
        - Cough characteristics (frequency, productivity)
        - Sputum changes (color, volume)
        - Fatigue level
        - Chest tightness
        - Wheezing
        """
        
    def generate_cat_score(responses: dict) -> int:
        """
        Calculates CAT score from voice responses:
        - 8 questions, 0-5 scale each
        - Total score 0-40
        """
        
    def assign_icd10_codes(symptoms: COPDSymptoms) -> list[str]:
        """
        Assigns appropriate ICD-10 codes:
        - J44.0: COPD with acute lower respiratory infection
        - J44.1: COPD with acute exacerbation
        - J44.9: COPD, unspecified
        """
        
    def generate_cpt_codes(monitoring_time: int) -> list[str]:
        """
        Generates CPT codes for billing:
        - 99457: Remote physiologic monitoring, first 20 minutes
        - 99458: Each additional 20 minutes
        - 99091: Collection and interpretation of physiologic data
        """
```

**Clinical Note Template:**
```
COPD Remote Monitoring Note
Date: {date}
Patient: {patient_name}
MRN: {medical_record_number}

SUBJECTIVE:
{patient_reported_symptoms}
CAT Score: {cat_score}/40
mMRC Dyspnea Scale: {mmrc_scale}/4

OBJECTIVE:
SpO2: {spo2_mean}% (range: {spo2_min}-{spo2_max}%)
Heart Rate: {hr_mean} bpm
Respiratory Rate: {rr_mean} breaths/min
Activity Level: {daily_steps} steps/day
Rescue Inhaler Use: {rescue_count} times/day

ASSESSMENT:
Exacerbation Risk: {risk_level} ({risk_score}/100)
GOLD Stage: {gold_stage}
{clinical_interpretation}

PLAN:
{action_plan_steps}
{medication_adjustments}
{follow_up_recommendations}

ICD-10: {icd10_codes}
CPT: {cpt_codes}
```

### 8. Environmental Monitoring Integration

**Air Quality API Integration:**

```python
class EnvironmentalMonitor:
    def get_current_aqi(location: Location) -> AQIData:
        """
        Fetches current AQI from EPA AirNow API:
        - PM2.5, PM10, O3, NO2, SO2, CO levels
        - Overall AQI (0-500 scale)
        - Health recommendations
        """
        
    def get_aqi_forecast(location: Location, hours: int = 24) -> list[AQIData]:
        """Returns hourly AQI forecast"""
        
    def calculate_exposure_score(aqi_history: list[AQIData], 
                                 activity_level: ActivityLevel) -> float:
        """
        Calculates cumulative exposure:
        - Higher weight for outdoor activity during poor AQI
        - Tracks daily exposure burden
        """
        
    def generate_recommendations(aqi: AQIData, 
                                patient_risk: RiskLevel) -> list[str]:
        """
        Provides personalized recommendations:
        - AQI 0-50: Normal activity
        - AQI 51-100: Sensitive groups limit prolonged outdoor exertion
        - AQI 101-150: Reduce outdoor activity
        - AQI 151+: Avoid outdoor activity
        """
```

**Pollen and Allergen Tracking:**
```python
def get_pollen_levels(location: Location) -> PollenData:
    """
    Fetches pollen forecast:
    - Tree, grass, weed pollen
    - Mold spore count
    - 5-day forecast
    """
```

### 9. Alert Management System

**Alert Classification:**

```python
class AlertLevel(Enum):
    CRITICAL = 1    # Immediate action required
    WARNING = 2     # Attention needed within hours
    INFO = 3        # Informational, no immediate action

class Alert:
    level: AlertLevel
    title: str
    message: str
    action_steps: list[str]
    patient_id: str
    timestamp: datetime
    acknowledged: bool
    acknowledged_by: str
    acknowledged_at: datetime
```

**Alert Rules:**

```python
def evaluate_alerts(patient_data: PatientData) -> list[Alert]:
    """
    Evaluates conditions and generates alerts:
    
    CRITICAL:
    - SpO2 < 88% for > 2 minutes
    - Heart rate > 120 or < 40 bpm
    - Respiratory rate > 30 breaths/min
    - Risk score > 85
    - No device data for > 6 hours
    
    WARNING:
    - SpO2 88-90% sustained
    - Risk score 70-85
    - Rescue inhaler > 4 times/day
    - Medication adherence < 70%
    - Declining trend in activity
    
    INFO:
    - Battery < 20%
    - Scheduled medication reminder
    - Exercise goal achieved
    - Weekly report available
    """
```

**Notification Routing:**

```python
def route_alert(alert: Alert, patient: Patient) -> None:
    """
    Routes alerts based on level and preferences:
    
    CRITICAL:
    - Push notification to patient app
    - SMS to patient
    - Push to caregiver app
    - SMS to caregivers
    - Alert on clinical dashboard
    - Email to on-call provider
    - Escalate if not acknowledged in 15 min
    
    WARNING:
    - Push notification to patient app
    - Alert on clinical dashboard
    - Daily digest to caregivers
    
    INFO:
    - In-app notification only
    """
```


## Data Models

### Core Data Structures

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum

@dataclass
class VitalSigns:
    patient_id: str
    timestamp: datetime
    spo2: float              # 0-100%
    heart_rate: float        # bpm
    respiratory_rate: float  # breaths/min
    temperature: float       # Celsius
    activity_level: str      # sedentary, light, moderate, vigorous
    body_position: str       # supine, sitting, standing, walking
    
@dataclass
class SymptomReport:
    patient_id: str
    timestamp: datetime
    cat_score: int           # 0-40
    mmrc_scale: int          # 0-4
    dyspnea_severity: int    # 0-10
    cough_frequency: str     # none, occasional, frequent, constant
    sputum_color: str        # clear, white, yellow, green, blood-tinged
    sputum_volume: str       # none, small, moderate, large
    chest_tightness: int     # 0-10
    wheezing: bool
    fatigue_level: int       # 0-10
    sleep_quality: int       # 0-10
    notes: str
    
@dataclass
class MedicationEvent:
    patient_id: str
    timestamp: datetime
    medication_name: str
    medication_type: str     # maintenance, rescue, nebulizer, oxygen
    dose: str
    route: str               # inhaled, oral, nebulized
    adherence: bool          # was it taken as scheduled?
    
@dataclass
class InhalerEvent:
    patient_id: str
    timestamp: datetime
    inhaler_type: str        # LABA, LAMA, ICS, SABA, SAMA
    actuations: int
    technique_correct: bool  # if sensor-enabled inhaler
    
@dataclass
class ExacerbationEvent:
    patient_id: str
    start_date: datetime
    end_date: datetime
    severity: str            # mild, moderate, severe
    treatment: str           # self-managed, outpatient, ED, hospitalization
    antibiotics_used: bool
    steroids_used: bool
    hospitalization_days: int
    
@dataclass
class RiskScore:
    patient_id: str
    timestamp: datetime
    overall_score: float     # 0-100
    vital_signs_score: float
    symptoms_score: float
    medication_score: float
    environment_score: float
    risk_level: str          # low, medium, high
    confidence: float        # 0-1
    contributing_factors: list[str]
    
@dataclass
class PatientBaseline:
    patient_id: str
    established_date: datetime
    spo2_mean: float
    spo2_std: float
    hr_mean: float
    hr_std: float
    rr_mean: float
    rr_std: float
    daily_steps_mean: float
    cat_score_baseline: int
    fev1_percent_predicted: float
    gold_stage: int          # 1-4
    
@dataclass
class EnvironmentData:
    patient_id: str
    timestamp: datetime
    location: tuple[float, float]  # lat, lon
    aqi: int                 # 0-500
    pm25: float              # μg/m³
    pm10: float
    ozone: float
    temperature: float       # Celsius
    humidity: float          # %
    pollen_level: str        # low, moderate, high, very high
    
@dataclass
class ActionPlan:
    patient_id: str
    green_zone: GreenZoneActions    # Doing well
    yellow_zone: YellowZoneActions  # Getting worse
    red_zone: RedZoneActions        # Medical emergency
    emergency_contacts: list[Contact]
    
@dataclass
class GreenZoneActions:
    description: str = "Breathing is good, no cough or wheeze"
    actions: list[str] = [
        "Take maintenance medications as prescribed",
        "Continue regular exercise and activities",
        "Monitor symptoms daily"
    ]
    
@dataclass
class YellowZoneActions:
    description: str = "Increased symptoms: more cough, mucus, shortness of breath"
    actions: list[str] = [
        "Increase rescue inhaler use",
        "Start oral steroids if prescribed",
        "Contact healthcare provider within 24 hours",
        "Reduce physical activity",
        "Monitor symptoms closely"
    ]
    
@dataclass
class RedZoneActions:
    description: str = "Severe symptoms: very short of breath, can't do usual activities"
    actions: list[str] = [
        "Take rescue inhaler immediately",
        "Call 911 or go to emergency room",
        "Take emergency medications if prescribed",
        "Notify emergency contacts"
    ]
```

### Database Schema

**DynamoDB Tables:**

```
Table: copd_vital_signs
Partition Key: patient_id (String)
Sort Key: timestamp (Number, Unix timestamp)
Attributes:
  - spo2, heart_rate, respiratory_rate, temperature
  - activity_level, body_position
  - device_id
TTL: 90 days (then archived to S3)

Table: copd_symptoms
Partition Key: patient_id (String)
Sort Key: timestamp (Number)
Attributes:
  - cat_score, mmrc_scale, dyspnea_severity
  - cough_frequency, sputum_color, sputum_volume
  - chest_tightness, wheezing, fatigue_level
  - sleep_quality, notes

Table: copd_medications
Partition Key: patient_id (String)
Sort Key: timestamp (Number)
Attributes:
  - medication_name, medication_type, dose, route
  - adherence, scheduled_time

Table: copd_risk_scores
Partition Key: patient_id (String)
Sort Key: timestamp (Number)
Attributes:
  - overall_score, vital_signs_score, symptoms_score
  - medication_score, environment_score
  - risk_level, confidence, contributing_factors

Table: copd_patients
Partition Key: patient_id (String)
Attributes:
  - demographics (name, dob, mrn)
  - baseline (established_date, vital_signs_baseline)
  - gold_stage, fev1_percent_predicted
  - comorbidities
  - care_team (provider_id, caregiver_ids)
  - device_id
  - preferences (notification_settings)

Table: copd_exacerbations
Partition Key: patient_id (String)
Sort Key: start_date (Number)
Attributes:
  - end_date, severity, treatment
  - antibiotics_used, steroids_used
  - hospitalization_days
  - predicted (was it predicted by ML model?)
  - prediction_lead_time (days)

Table: copd_alerts
Partition Key: patient_id (String)
Sort Key: timestamp (Number)
Attributes:
  - alert_level, title, message, action_steps
  - acknowledged, acknowledged_by, acknowledged_at
  - resolved, resolved_at
GSI: alert_level-timestamp-index (for querying all critical alerts)
```

**S3 Bucket Structure:**

```
copd-data-{account-id}/
├── raw-sensor-data/
│   └── {patient_id}/
│       └── {year}/{month}/{day}/
│           └── {device_id}_{timestamp}.json
├── ml-training-data/
│   ├── features/
│   │   └── {cohort}_{date}.parquet
│   └── labels/
│       └── exacerbations_{date}.parquet
├── ml-models/
│   └── exacerbation-predictor/
│       └── {version}/
│           ├── model.tar.gz
│           └── metadata.json
├── clinical-notes/
│   └── {patient_id}/
│       └── {year}/{month}/
│           └── note_{timestamp}.pdf
└── archived-data/
    └── {patient_id}/
        └── {year}/
            └── vitals_{month}.parquet
```


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Vital Signs Monitoring Properties

**Property 1: SpO2 Measurement Accuracy**
*For any* true SpO2 value between 70-100%, the measured SpO2 value should be within ±2% of the true value.
**Validates: Requirements 1.1**

**Property 2: Low Oxygen Alert Generation**
*For any* time series of SpO2 measurements, when SpO2 remains below 88% for more than 2 minutes, an alert should be generated within the monitoring period.
**Validates: Requirements 1.2**

**Property 3: Respiratory Rate Measurement Accuracy**
*For any* true respiratory rate, the measured respiratory rate should be within ±2 breaths per minute of the true value.
**Validates: Requirements 1.3**

**Property 4: Heart Rate Measurement Accuracy**
*For any* true heart rate, the measured heart rate should be within ±3 beats per minute of the true value.
**Validates: Requirements 1.4**

**Property 5: Vital Signs Timestamp Accuracy**
*For any* vital sign measurement, the stored timestamp should be within 1 second of the actual measurement time.
**Validates: Requirements 1.9**

**Property 6: Data Transmission Interval**
*For any* sequence of data transmissions to AWS IoT Core, the maximum interval between consecutive transmissions should not exceed 5 minutes.
**Validates: Requirements 1.10**

### Exacerbation Detection Properties

**Property 7: Exacerbation Prediction Lead Time**
*For any* predicted exacerbation that actually occurs, the prediction should have been made 3-7 days before the exacerbation event.
**Validates: Requirements 2.1**

**Property 8: High Risk Notification Timing**
*For any* high risk detection event, notifications to patient and provider should be sent within 5 minutes of detection.
**Validates: Requirements 2.2**

**Property 9: CAT Score Calculation**
*For any* set of 8 symptom responses (each 0-5), the calculated CAT score should equal the sum of responses and be in range 0-40.
**Validates: Requirements 2.3**

**Property 10: Risk Score Classification**
*For any* calculated risk score, if the score exceeds 70, the patient should be classified as high risk.
**Validates: Requirements 2.6**

**Property 11: Exacerbation Predictor Sensitivity**
*For any* validation dataset with labeled exacerbations, the model should correctly identify at least 80% of actual exacerbations (sensitivity ≥ 0.80).
**Validates: Requirements 2.7**

**Property 12: Exacerbation Predictor Specificity**
*For any* validation dataset with labeled non-exacerbations, the model should correctly identify at least 75% of non-exacerbation cases (specificity ≥ 0.75).
**Validates: Requirements 2.8**

**Property 13: Baseline Establishment Timing**
*For any* new patient, personalized baseline values should be established within 14 days of initial device use.
**Validates: Requirements 2.9**

**Property 14: Anomaly Detection Threshold**
*For any* vital sign measurement and established baseline, an anomaly should be detected when the measurement deviates more than 2 standard deviations from the baseline mean.
**Validates: Requirements 2.10**

### Medication Management Properties

**Property 15: Inhaler Event Recording**
*For any* inhaler actuation, the event should be recorded with a timestamp accurate to within 1 second.
**Validates: Requirements 3.1**

**Property 16: Inhaler Type Classification**
*For any* recorded inhaler event, the inhaler should be correctly classified as either maintenance or rescue type.
**Validates: Requirements 3.2**

**Property 17: Missed Medication Reminder Timing**
*For any* scheduled medication dose that is not taken within the scheduled window, a reminder should be sent within 15 minutes of the missed time.
**Validates: Requirements 3.3**

**Property 18: Medication Adherence Calculation**
*For any* day with scheduled medications, adherence percentage should equal (doses taken / doses scheduled) × 100.
**Validates: Requirements 3.4**

**Property 19: Rescue Inhaler Alert Threshold**
*For any* day where rescue inhaler usage exceeds 4 actuations, an alert should be generated.
**Validates: Requirements 3.5**

**Property 20: Daily Adherence Data Transmission**
*For any* patient, medication adherence data should be transmitted to the Clinical Dashboard at least once per day.
**Validates: Requirements 3.10**

### Environmental Monitoring Properties

**Property 21: AQI Recommendation Threshold**
*For any* location where AQI exceeds 100, the Patient App should recommend limiting outdoor activity.
**Validates: Requirements 4.2**

**Property 22: Poor Air Quality Alert Lead Time**
*For any* predicted poor air quality event, an alert should be sent to the patient at least 2 hours before the predicted event.
**Validates: Requirements 4.10**

**Property 23: Weekly Exposure Report Generation**
*For any* patient, a personalized environmental exposure report should be generated weekly.
**Validates: Requirements 4.9**

### Patient Engagement Properties

**Property 24: Daily Exercise Recommendations**
*For any* patient, pulmonary rehabilitation exercise recommendations should be provided daily.
**Validates: Requirements 5.1**

**Property 25: Exercise Session Recording**
*For any* completed exercise session, the duration and intensity should be recorded.
**Validates: Requirements 5.3**

**Property 26: Medication Reminder Timing**
*For any* scheduled medication, a reminder should be sent at the scheduled time (within 1 minute tolerance).
**Validates: Requirements 5.6**

**Property 27: Action Plan Guidance Trigger**
*For any* symptom worsening event, the Patient App should guide the patient through their personalized Action Plan.
**Validates: Requirements 5.8**

**Property 28: Goal Achievement Reinforcement**
*For any* rehabilitation goal that is achieved, positive reinforcement should be provided to the patient.
**Validates: Requirements 5.10**

### Clinical Integration Properties

**Property 29: FHIR Export Round Trip**
*For any* patient data exported in HL7 FHIR format, importing the exported data should produce equivalent patient records.
**Validates: Requirements 6.6**

**Property 30: GOLD Stage Display**
*For any* patient with spirometry data, the Clinical Dashboard should display the correct GOLD Stage classification based on FEV1 % predicted.
**Validates: Requirements 6.2**

### AWS HealthScribe Properties

**Property 31: Voice Symptom Transcription**
*For any* voice recording of symptom reports, AWS HealthScribe should produce a text transcript and structured symptom data.
**Validates: Requirements 7.2**

**Property 32: Clinical Note Structure**
*For any* generated clinical note, it should contain all required sections: Subjective, Objective, Assessment, and Plan.
**Validates: Requirements 7.3**

**Property 33: COPD Symptom Extraction**
*For any* transcript containing mentions of dyspnea, cough, or sputum changes, these symptoms should be extracted into structured fields.
**Validates: Requirements 7.4**

**Property 34: ICD-10 Code Assignment**
*For any* documented COPD exacerbation, appropriate ICD-10 codes (J44.0 or J44.1) should be assigned.
**Validates: Requirements 7.5**

**Property 35: CPT Code Generation**
*For any* remote monitoring session, appropriate CPT codes (99457, 99458, or 99091) should be generated based on monitoring duration.
**Validates: Requirements 7.6**

**Property 36: CAT Score from Voice**
*For any* voice-based CAT assessment with 8 responses, the calculated score should equal the sum of response values (0-40 range).
**Validates: Requirements 7.7**

### Machine Learning Properties

**Property 37: Prediction Feature Inclusion**
*For any* exacerbation prediction, the model should incorporate vital signs, symptoms, medication adherence, and environmental data as input features.
**Validates: Requirements 8.2**

**Property 38: Monthly Model Updates**
*For any* patient, the prediction model should be updated with new data at least once per month.
**Validates: Requirements 8.3**

**Property 39: Baseline Calculation Period**
*For any* patient, baseline values should be calculated using data from the first 14 days of device use.
**Validates: Requirements 8.4**

**Property 40: Risk Category Classification**
*For any* risk score, it should be classified as: low (0-33), medium (34-66), or high (67-100).
**Validates: Requirements 8.6**

**Property 41: Prediction Confidence Scores**
*For any* exacerbation prediction, a confidence score in the range [0, 1] should be provided.
**Validates: Requirements 8.7**

**Property 42: Model Retraining Trigger**
*For any* prediction model, when accuracy falls below 75%, retraining should be triggered.
**Validates: Requirements 8.9**

### Caregiver Integration Properties

**Property 43: Critical Alert Caregiver Notification**
*For any* critical alert, designated caregivers should be notified within 2 minutes of alert generation.
**Validates: Requirements 9.2**

**Property 44: Patient Access Control**
*For any* caregiver, access to patient data should only be granted if the patient has authorized that caregiver.
**Validates: Requirements 9.6**

**Property 45: Daily Caregiver Summary**
*For any* authorized caregiver, a daily summary report should be sent.
**Validates: Requirements 9.7**

**Property 46: Caregiver Note Visibility**
*For any* note added by a caregiver, it should be visible to the patient's healthcare team.
**Validates: Requirements 9.8**

**Property 47: Permission Level Enforcement**
*For any* caregiver with a specific permission level, access should be restricted to only the data and functions allowed by that permission level.
**Validates: Requirements 9.9**

### Security and Compliance Properties

**Property 48: Data Encryption**
*For any* stored patient health information, it should be encrypted using AES-256 encryption.
**Validates: Requirements 10.2**

**Property 49: Access Control Enforcement**
*For any* attempt to access PHI without proper authorization, access should be denied.
**Validates: Requirements 10.3**

**Property 50: Audit Log Retention**
*For any* data access event, an audit log entry should be created and retained for at least 6 years.
**Validates: Requirements 10.4**

**Property 51: Patient Data Rights**
*For any* patient request to access, modify, or delete their data, the system should support the requested operation.
**Validates: Requirements 10.5**

**Property 52: Multi-Factor Authentication**
*For any* user login attempt, multi-factor authentication should be required before granting access.
**Validates: Requirements 10.8**

### Alert Management Properties

**Property 53: Alert Priority Classification**
*For any* generated alert, it should be assigned exactly one priority level: critical, warning, or informational.
**Validates: Requirements 11.1**

**Property 54: Critical Alert Multi-Recipient Notification**
*For any* critical alert, the patient, all authorized caregivers, and the healthcare provider should all receive notifications.
**Validates: Requirements 11.2**

**Property 55: Duplicate Alert Suppression**
*For any* alert condition, if an alert was already sent for that condition, no duplicate alert should be sent within 2 hours.
**Validates: Requirements 11.3**

**Property 56: Alert Acknowledgment Recording**
*For any* acknowledged alert, the acknowledgment should be recorded with a timestamp and the acknowledging user's identity.
**Validates: Requirements 11.5**

**Property 57: Critical Alert Escalation**
*For any* critical alert that remains unacknowledged, escalation should occur after 15 minutes.
**Validates: Requirements 11.6**

**Property 58: Alert Action Steps**
*For any* generated alert, it should include at least one clear action step.
**Validates: Requirements 11.7**

### Data Storage Properties

**Property 59: Data Retrieval Latency**
*For any* set of 100 patient data retrieval requests, at least 95 should complete with latency less than 500ms.
**Validates: Requirements 12.3**

**Property 60: Historical Data Compression**
*For any* data older than 90 days, it should be stored in compressed format.
**Validates: Requirements 12.5**

**Property 61: Daily Backup Creation**
*For any* day, at least one backup snapshot should be created.
**Validates: Requirements 12.7**

**Property 62: Data Export Round Trip**
*For any* patient data exported in CSV or JSON format, importing the exported data should produce equivalent patient records.
**Validates: Requirements 12.8**

**Property 63: Data Integrity Validation**
*For any* stored data, checksums should be calculated and validated to ensure integrity.
**Validates: Requirements 12.10**

### Device Management Properties

**Property 64: Hourly Battery Reporting**
*For any* 1-hour time period, at least one battery level report should be sent to AWS IoT Core.
**Validates: Requirements 13.1**

**Property 65: Low Battery Reminder**
*For any* battery level measurement below 20%, a charging reminder should be displayed in the Patient App.
**Validates: Requirements 13.2**

**Property 66: Daily Health Metrics Reporting**
*For any* day, firmware version and device health metrics should be reported at least once.
**Validates: Requirements 13.3**

**Property 67: Offline Data Buffering**
*For any* period of lost connectivity up to 24 hours, all generated data should be buffered locally.
**Validates: Requirements 13.6**

**Property 68: Automatic Reconnection**
*For any* connectivity loss event, when connectivity is restored, the device should automatically reconnect.
**Validates: Requirements 13.7**

**Property 69: Monthly Calibration Reporting**
*For any* 30-day period, at least one sensor calibration status report should be sent.
**Validates: Requirements 13.8**

### Reporting and Analytics Properties

**Property 70: Monthly Progress Reports**
*For any* patient, a progress report should be generated at least once per month.
**Validates: Requirements 14.1**

**Property 71: Exacerbation Frequency Calculation**
*For any* patient and time period, exacerbation frequency should equal the count of exacerbation events divided by the number of days in the period.
**Validates: Requirements 14.4**

**Property 72: PDF Report Export**
*For any* generated report, it should be exportable in PDF format.
**Validates: Requirements 14.7**

**Property 73: High-Risk Patient Highlighting**
*For any* patient with risk score > 70, they should be highlighted in the Clinical Dashboard patient list.
**Validates: Requirements 14.8**


## Error Handling

### Device-Level Error Handling

**Sensor Failures:**
```python
class SensorErrorHandler:
    def handle_sensor_failure(sensor_type: str, error: Exception) -> None:
        """
        Handles sensor failures:
        1. Log error with timestamp and sensor details
        2. Attempt sensor reset (up to 3 retries)
        3. If reset fails, mark sensor as failed
        4. Generate device health alert
        5. Continue operation with remaining sensors
        6. Notify technical support if failure persists > 1 hour
        """
        
    def validate_sensor_reading(reading: float, sensor_type: str) -> bool:
        """
        Validates sensor readings:
        - Check for out-of-range values
        - Detect impossible physiological values
        - Identify sensor drift or calibration issues
        - Return False for invalid readings
        """
```

**Connectivity Errors:**
```python
class ConnectivityErrorHandler:
    def handle_connection_loss() -> None:
        """
        Handles connectivity loss:
        1. Enable local data buffering
        2. Attempt reconnection with exponential backoff
        3. Try alternative connectivity methods (Wi-Fi → Cellular → Bluetooth)
        4. Display connectivity status to user
        5. Alert user if offline > 6 hours
        """
        
    def handle_transmission_failure(data: SensorData) -> None:
        """
        Handles failed data transmission:
        1. Add data to retry queue
        2. Attempt retransmission up to 5 times
        3. If all retries fail, persist to local storage
        4. Transmit when connectivity restored
        """
```

**Battery and Power Errors:**
```python
class PowerErrorHandler:
    def handle_low_battery(level: float) -> None:
        """
        Handles low battery conditions:
        - level < 20%: Display charging reminder
        - level < 10%: Reduce sampling frequency
        - level < 5%: Critical alert, emergency data save
        - level < 2%: Graceful shutdown with data preservation
        """
        
    def handle_charging_error() -> None:
        """
        Handles charging failures:
        1. Detect charging anomalies
        2. Alert user to check charging connection
        3. Log charging error for diagnostics
        """
```

### Backend Error Handling

**Data Processing Errors:**
```python
class DataProcessingErrorHandler:
    def handle_invalid_data(data: dict, validation_error: Exception) -> None:
        """
        Handles invalid incoming data:
        1. Log validation error with data sample
        2. Send data to dead-letter queue for analysis
        3. Alert monitoring system
        4. Do not process invalid data
        5. Continue processing valid data
        """
        
    def handle_processing_timeout(patient_id: str, data: dict) -> None:
        """
        Handles processing timeouts:
        1. Log timeout event
        2. Retry processing with increased timeout
        3. If retry fails, escalate to manual review
        4. Ensure no data loss
        """
```

**ML Model Errors:**
```python
class MLErrorHandler:
    def handle_prediction_failure(patient_id: str, error: Exception) -> None:
        """
        Handles ML prediction failures:
        1. Log error with patient context
        2. Fall back to rule-based risk scoring
        3. Alert ML ops team
        4. Continue monitoring with fallback method
        5. Do not interrupt patient care
        """
        
    def handle_model_drift(metrics: ModelMetrics) -> None:
        """
        Handles model performance degradation:
        1. Detect accuracy drop below threshold
        2. Trigger model retraining
        3. A/B test new model vs current model
        4. Roll back if new model performs worse
        """
```

**API and Integration Errors:**
```python
class IntegrationErrorHandler:
    def handle_healthscribe_error(error: Exception) -> None:
        """
        Handles AWS HealthScribe failures:
        1. Log error with request details
        2. Retry with exponential backoff (max 3 retries)
        3. If all retries fail, store audio for manual transcription
        4. Alert clinical team of transcription delay
        """
        
    def handle_external_api_failure(api_name: str, error: Exception) -> None:
        """
        Handles external API failures (AQI, weather, etc.):
        1. Use cached data if available
        2. Retry with timeout
        3. If unavailable, disable dependent features temporarily
        4. Alert user that feature is temporarily unavailable
        5. Resume when API is available
        """
```

**Database Errors:**
```python
class DatabaseErrorHandler:
    def handle_write_failure(data: dict, error: Exception) -> None:
        """
        Handles database write failures:
        1. Log error with data payload
        2. Retry write operation (max 3 attempts)
        3. If all retries fail, write to backup storage (S3)
        4. Alert database team
        5. Attempt to replay from backup when database recovers
        """
        
    def handle_read_failure(query: dict, error: Exception) -> None:
        """
        Handles database read failures:
        1. Retry read operation
        2. Try read replica if primary fails
        3. Return cached data if available
        4. Return error to user with retry option
        5. Alert database team
        """
```

### Alert and Notification Errors

```python
class NotificationErrorHandler:
    def handle_notification_failure(alert: Alert, recipient: str, error: Exception) -> None:
        """
        Handles notification delivery failures:
        1. Log failure with recipient and alert details
        2. Try alternative notification channels (push → SMS → email)
        3. If all channels fail, log for manual follow-up
        4. For critical alerts, escalate to on-call provider
        5. Retry notification when connectivity restored
        """
        
    def handle_alert_storm(patient_id: str, alert_count: int) -> None:
        """
        Handles excessive alert generation:
        1. Detect > 10 alerts in 1 hour
        2. Suppress non-critical alerts
        3. Consolidate similar alerts
        4. Alert clinical team of potential device malfunction
        5. Investigate root cause
        """
```

### User-Facing Error Messages

**Error Message Principles:**
- Clear, non-technical language for patients
- Specific, actionable guidance
- Avoid alarming language
- Provide next steps
- Include support contact information

**Example Error Messages:**

```
Connectivity Issue:
"We're having trouble connecting to the internet. Your device is still 
monitoring your health and will send data when connection is restored. 
If this continues for more than 6 hours, please contact support."

Sensor Issue:
"We're having trouble reading your oxygen level. Please make sure the 
device is positioned correctly on your wrist. If the problem continues, 
contact support for assistance."

Low Battery:
"Battery is low (15%). Please charge your device soon to ensure 
continuous monitoring."

Data Sync Issue:
"Some of your health data hasn't synced yet. We'll automatically sync 
when connection improves. Your device is still monitoring your health."
```


## Testing Strategy

### Dual Testing Approach

The RTI Kit for COPD requires comprehensive testing using both unit tests and property-based tests. These approaches are complementary and together provide thorough coverage:

- **Unit Tests**: Verify specific examples, edge cases, error conditions, and integration points
- **Property-Based Tests**: Verify universal properties across all inputs through randomization

Unit tests should focus on concrete scenarios and edge cases, while property-based tests validate that correctness properties hold across the entire input space. Together, they catch both specific bugs and general correctness violations.

### Property-Based Testing Configuration

**Testing Library Selection:**
- **Python**: Use Hypothesis for property-based testing
- **TypeScript/JavaScript**: Use fast-check for property-based testing
- **Java**: Use jqwik for property-based testing

**Test Configuration:**
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property
- Tag format: `# Feature: rti-kit-copd, Property {number}: {property_text}`

**Property Test Implementation:**
Each correctness property from the design document must be implemented as a single property-based test. The test should:
1. Generate random valid inputs
2. Execute the system under test
3. Assert the property holds for all generated inputs
4. Reference the property number from the design document

### Testing Scope by Component

#### 1. Wearable Device Firmware Testing

**Unit Tests:**
- Sensor initialization and calibration
- Data collection at specified intervals
- Local data buffering when offline
- Battery management state transitions
- Firmware update process
- Error recovery procedures

**Property Tests:**
```python
# Feature: rti-kit-copd, Property 1: SpO2 Measurement Accuracy
@given(st.floats(min_value=70, max_value=100))
def test_spo2_measurement_accuracy(true_spo2):
    measured_spo2 = device.read_spo2()
    assert abs(measured_spo2 - true_spo2) <= 2.0

# Feature: rti-kit-copd, Property 3: Respiratory Rate Measurement Accuracy
@given(st.floats(min_value=8, max_value=40))
def test_respiratory_rate_accuracy(true_rr):
    measured_rr = device.read_respiratory_rate()
    assert abs(measured_rr - true_rr) <= 2.0

# Feature: rti-kit-copd, Property 6: Data Transmission Interval
@given(st.lists(st.datetimes(), min_size=2))
def test_transmission_interval(timestamps):
    intervals = [timestamps[i+1] - timestamps[i] for i in range(len(timestamps)-1)]
    max_interval = max(intervals)
    assert max_interval.total_seconds() <= 300  # 5 minutes
```

#### 2. Data Processing Pipeline Testing

**Unit Tests:**
- Lambda function input validation
- Data transformation correctness
- Error handling for malformed data
- Step Functions workflow transitions
- DynamoDB write operations
- S3 archival processes

**Property Tests:**
```python
# Feature: rti-kit-copd, Property 9: CAT Score Calculation
@given(st.lists(st.integers(min_value=0, max_value=5), min_size=8, max_size=8))
def test_cat_score_calculation(responses):
    cat_score = calculate_cat_score(responses)
    assert cat_score == sum(responses)
    assert 0 <= cat_score <= 40

# Feature: rti-kit-copd, Property 18: Medication Adherence Calculation
@given(st.integers(min_value=0, max_value=10), st.integers(min_value=0, max_value=10))
def test_adherence_calculation(doses_taken, doses_scheduled):
    if doses_scheduled == 0:
        assert calculate_adherence(doses_taken, doses_scheduled) == 100.0
    else:
        expected = (doses_taken / doses_scheduled) * 100
        assert calculate_adherence(doses_taken, doses_scheduled) == expected

# Feature: rti-kit-copd, Property 40: Risk Category Classification
@given(st.floats(min_value=0, max_value=100))
def test_risk_category_classification(risk_score):
    category = classify_risk(risk_score)
    if 0 <= risk_score <= 33:
        assert category == "low"
    elif 34 <= risk_score <= 66:
        assert category == "medium"
    else:
        assert category == "high"
```

#### 3. Machine Learning Model Testing

**Unit Tests:**
- Feature engineering correctness
- Model input validation
- Prediction output format
- Model versioning and rollback
- A/B testing infrastructure

**Property Tests:**
```python
# Feature: rti-kit-copd, Property 11: Exacerbation Predictor Sensitivity
@given(validation_dataset=st.data())
def test_model_sensitivity(validation_dataset):
    predictions = model.predict(validation_dataset.features)
    true_positives = sum((p == 1 and l == 1) for p, l in zip(predictions, validation_dataset.labels))
    actual_positives = sum(validation_dataset.labels)
    sensitivity = true_positives / actual_positives if actual_positives > 0 else 1.0
    assert sensitivity >= 0.80

# Feature: rti-kit-copd, Property 12: Exacerbation Predictor Specificity
@given(validation_dataset=st.data())
def test_model_specificity(validation_dataset):
    predictions = model.predict(validation_dataset.features)
    true_negatives = sum((p == 0 and l == 0) for p, l in zip(predictions, validation_dataset.labels))
    actual_negatives = len(validation_dataset.labels) - sum(validation_dataset.labels)
    specificity = true_negatives / actual_negatives if actual_negatives > 0 else 1.0
    assert specificity >= 0.75

# Feature: rti-kit-copd, Property 41: Prediction Confidence Scores
@given(st.data())
def test_prediction_confidence_range(patient_data):
    prediction, confidence = model.predict_with_confidence(patient_data)
    assert 0.0 <= confidence <= 1.0
```

#### 4. Alert Management Testing

**Unit Tests:**
- Alert priority assignment logic
- Notification routing rules
- Alert suppression timing
- Escalation procedures
- Alert acknowledgment workflow

**Property Tests:**
```python
# Feature: rti-kit-copd, Property 53: Alert Priority Classification
@given(st.data())
def test_alert_priority_classification(alert_data):
    alert = generate_alert(alert_data)
    assert alert.level in [AlertLevel.CRITICAL, AlertLevel.WARNING, AlertLevel.INFO]

# Feature: rti-kit-copd, Property 55: Duplicate Alert Suppression
@given(st.lists(st.datetimes(), min_size=2))
def test_duplicate_alert_suppression(alert_timestamps):
    alerts = [Alert(timestamp=ts, condition="low_spo2") for ts in alert_timestamps]
    sent_alerts = filter_duplicate_alerts(alerts, suppression_window=timedelta(hours=2))
    for i in range(len(sent_alerts) - 1):
        time_diff = sent_alerts[i+1].timestamp - sent_alerts[i].timestamp
        assert time_diff >= timedelta(hours=2)

# Feature: rti-kit-copd, Property 57: Critical Alert Escalation
@given(st.integers(min_value=0, max_value=30))
def test_critical_alert_escalation(minutes_unacknowledged):
    alert = Alert(level=AlertLevel.CRITICAL, timestamp=datetime.now())
    should_escalate = check_escalation(alert, minutes_unacknowledged)
    if minutes_unacknowledged > 15:
        assert should_escalate == True
    else:
        assert should_escalate == False
```

#### 5. AWS HealthScribe Integration Testing

**Unit Tests:**
- Audio upload to S3
- HealthScribe API invocation
- Response parsing
- Clinical note generation
- ICD-10 and CPT code assignment

**Property Tests:**
```python
# Feature: rti-kit-copd, Property 32: Clinical Note Structure
@given(st.data())
def test_clinical_note_structure(symptom_data):
    note = generate_clinical_note(symptom_data)
    assert "SUBJECTIVE:" in note
    assert "OBJECTIVE:" in note
    assert "ASSESSMENT:" in note
    assert "PLAN:" in note

# Feature: rti-kit-copd, Property 36: CAT Score from Voice
@given(st.lists(st.integers(min_value=0, max_value=5), min_size=8, max_size=8))
def test_cat_score_from_voice(voice_responses):
    transcript = generate_voice_transcript(voice_responses)
    extracted_score = extract_cat_score(transcript)
    assert extracted_score == sum(voice_responses)
    assert 0 <= extracted_score <= 40
```

#### 6. Data Storage and Retrieval Testing

**Unit Tests:**
- DynamoDB table operations
- S3 bucket operations
- Data partitioning logic
- TTL and archival processes
- Backup and restore procedures

**Property Tests:**
```python
# Feature: rti-kit-copd, Property 59: Data Retrieval Latency
@given(st.lists(st.text(), min_size=100, max_size=100))
def test_data_retrieval_latency(patient_ids):
    latencies = []
    for patient_id in patient_ids:
        start = time.time()
        data = retrieve_patient_data(patient_id)
        latency = time.time() - start
        latencies.append(latency)
    
    p95_latency = sorted(latencies)[94]  # 95th percentile
    assert p95_latency < 0.5  # 500ms

# Feature: rti-kit-copd, Property 62: Data Export Round Trip
@given(st.data())
def test_data_export_round_trip(patient_data):
    # Export to CSV
    csv_export = export_to_csv(patient_data)
    imported_from_csv = import_from_csv(csv_export)
    assert imported_from_csv == patient_data
    
    # Export to JSON
    json_export = export_to_json(patient_data)
    imported_from_json = import_from_json(json_export)
    assert imported_from_json == patient_data

# Feature: rti-kit-copd, Property 63: Data Integrity Validation
@given(st.data())
def test_data_integrity(patient_data):
    stored_data = store_with_checksum(patient_data)
    retrieved_data = retrieve_with_validation(stored_data.id)
    assert validate_checksum(retrieved_data) == True
    assert retrieved_data.data == patient_data
```

#### 7. Security and Access Control Testing

**Unit Tests:**
- Authentication flows
- Authorization rules
- Encryption/decryption
- Audit logging
- Session management

**Property Tests:**
```python
# Feature: rti-kit-copd, Property 49: Access Control Enforcement
@given(st.data())
def test_access_control_enforcement(access_attempt):
    user = access_attempt.user
    resource = access_attempt.resource
    
    if not user.is_authorized_for(resource):
        with pytest.raises(UnauthorizedError):
            access_resource(user, resource)

# Feature: rti-kit-copd, Property 48: Data Encryption
@given(st.binary())
def test_data_encryption(phi_data):
    encrypted = encrypt_phi(phi_data)
    assert encrypted != phi_data  # Data is transformed
    assert get_encryption_algorithm(encrypted) == "AES-256"
    
    decrypted = decrypt_phi(encrypted)
    assert decrypted == phi_data  # Round trip works

# Feature: rti-kit-copd, Property 50: Audit Log Retention
@given(st.lists(st.datetimes(), min_size=1))
def test_audit_log_retention(access_events):
    for event in access_events:
        log_access(event)
    
    six_years_later = datetime.now() + timedelta(days=365*6)
    retained_logs = get_audit_logs(up_to=six_years_later)
    assert len(retained_logs) == len(access_events)
```

### Integration Testing

**End-to-End Scenarios:**
1. Device data collection → AWS IoT → Processing → Storage → Dashboard display
2. Symptom reporting → HealthScribe → Clinical note → Provider notification
3. Exacerbation prediction → Alert generation → Multi-channel notification
4. Medication reminder → Patient acknowledgment → Adherence calculation
5. Environmental trigger → Activity recommendation → Patient compliance

**Integration Test Examples:**
```python
def test_exacerbation_detection_workflow():
    # Simulate declining vital signs
    patient = create_test_patient()
    for day in range(7):
        vitals = generate_declining_vitals(day)
        device.send_vitals(patient.id, vitals)
    
    # Verify prediction was made
    predictions = get_predictions(patient.id)
    assert len(predictions) > 0
    assert predictions[-1].risk_level == "high"
    
    # Verify alerts were sent
    alerts = get_alerts(patient.id)
    assert any(a.level == AlertLevel.CRITICAL for a in alerts)
    
    # Verify notifications were delivered
    notifications = get_notifications(patient.id)
    assert any(n.recipient == patient.provider_id for n in notifications)
    assert any(n.recipient in patient.caregiver_ids for n in notifications)

def test_medication_adherence_workflow():
    patient = create_test_patient()
    schedule = create_medication_schedule(patient.id, times=["08:00", "20:00"])
    
    # Simulate missed morning dose
    advance_time_to("08:16")
    reminders = get_reminders(patient.id)
    assert len(reminders) == 1
    assert "medication" in reminders[0].message.lower()
    
    # Simulate taking evening dose
    advance_time_to("20:00")
    device.record_medication(patient.id, "maintenance_inhaler")
    
    # Verify adherence calculation
    adherence = calculate_daily_adherence(patient.id)
    assert adherence == 50.0  # 1 of 2 doses taken
```

### Performance Testing

**Load Testing:**
- Simulate 10,000 concurrent devices sending data
- Verify system handles peak load without degradation
- Measure end-to-end latency under load

**Stress Testing:**
- Test system behavior at 2x expected load
- Identify breaking points
- Verify graceful degradation

**Endurance Testing:**
- Run system for 7 days continuous operation
- Monitor for memory leaks
- Verify data consistency over time

### Clinical Validation Testing

**Accuracy Validation:**
- Compare device measurements against gold standard equipment
- Validate SpO2 accuracy across range 70-100%
- Validate respiratory rate and heart rate accuracy
- Test with diverse patient populations

**Exacerbation Prediction Validation:**
- Retrospective analysis on historical patient data
- Prospective validation study with real patients
- Measure sensitivity, specificity, PPV, NPV
- Calculate lead time for predictions

**Usability Testing:**
- Test with actual COPD patients
- Measure task completion rates
- Gather user feedback
- Iterate on UI/UX based on findings

### Regulatory Testing

**FDA Compliance:**
- Software verification and validation (V&V)
- Risk analysis (ISO 14971)
- Cybersecurity testing (FDA guidance)
- Clinical evaluation report

**HIPAA Compliance:**
- Security risk assessment
- Penetration testing
- Audit log review
- Breach notification procedures testing

