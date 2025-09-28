# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.0
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

### Roomba Adapter Specific Context

This is the **ioBroker.roomba** adapter that connects iRobot Roomba vacuum cleaners to the ioBroker smart home platform. Key aspects:

- **Primary Function**: Bridge between iRobot Roomba devices and ioBroker using the unofficial dorita980 API
- **Key Dependencies**: 
  - `@karlvr/dorita980`: Unofficial Roomba API library for device communication
  - `canvas`: Map drawing and visualization capabilities
  - `@iobroker/adapter-core`: Standard ioBroker adapter framework
- **Main Features**:
  - Remote control (start, stop, pause, resume, dock commands)
  - Real-time status monitoring (battery, dock status, bin status)
  - Mission tracking with statistics and runtime data
  - Map visualization and drawing from mission data
  - Web interface for status display and historical mission data
  - Device configuration retrieval (preferences, network, schedule)
- **Authentication**: Uses WiFi credential retrieval process (different from mobile app credentials)
- **Supported Models**: Various Roomba series (600, 700, 800, 900, i-series, s-series) with different firmware versions
- **Map Support**: Available on newer models that provide positioning data during missions

## Testing

### Unit Testing
- Use Jest as the primary testing framework for ioBroker adapters
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files to allow testing of functionality without live connections
- Example test structure:
  ```javascript
  describe('AdapterName', () => {
    let adapter;
    
    beforeEach(() => {
      // Setup test adapter instance
    });
    
    test('should initialize correctly', () => {
      // Test adapter initialization
    });
  });
  ```

### Integration Testing

**IMPORTANT**: Use the official `@iobroker/testing` framework for all integration tests. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation**: https://github.com/ioBroker/testing

#### Framework Structure
Integration tests MUST follow this exact pattern:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

// Define test coordinates or configuration
const TEST_COORDINATES = '52.520008,13.404954'; // Berlin
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();
                        
                        // Get adapter object using promisified pattern
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) {
                            return reject(new Error('Adapter object not found'));
                        }

                        // Configure adapter properties
                        Object.assign(obj.native, {
                            position: TEST_COORDINATES,
                            createCurrently: true,
                            createHourly: true,
                            createDaily: true,
                            // Add other configuration as needed
                        });

                        // Set the updated configuration
                        harness.objects.setObject(obj._id, obj);

                        console.log('✅ Step 1: Configuration written, starting adapter...');
                        
                        // Start adapter and wait
                        await harness.startAdapterAndWait();
                        
                        console.log('✅ Step 2: Adapter started');

                        // Wait for adapter to process data
                        const waitMs = 15000;
                        await wait(waitMs);

                        console.log('🔍 Step 3: Checking states after adapter run...');
                        
                        // Get all states created by adapter
                        const stateIds = await harness.dbConnection.getStateIDs('your-adapter.0.*');
                        
                        console.log(`📊 Found ${stateIds.length} states`);

                        if (stateIds.length > 0) {
                            console.log('✅ Adapter successfully created states');
                            
                            // Show sample of created states
                            const allStates = await new Promise((res, rej) => {
                                harness.states.getStates(stateIds, (err, states) => {
                                    if (err) return rej(err);
                                    res(states || []);
                                });
                            });
                            
                            console.log('📋 Sample states created:');
                            stateIds.slice(0, 5).forEach((stateId, index) => {
                                const state = allStates[index];
                                console.log(`   ${stateId}: ${state && state.val !== undefined ? state.val : 'undefined'}`);
                            });
                            
                            await harness.stopAdapter();
                            resolve(true);
                        } else {
                            console.log('❌ No states were created by the adapter');
                            reject(new Error('Adapter did not create any states'));
                        }
                    } catch (error) {
                        reject(error);
                    }
                });
            }).timeout(40000);
        });
    }
});
```

#### Testing Both Success AND Failure Scenarios

**IMPORTANT**: For every "it works" test, implement corresponding "it doesn't work and fails" tests. This ensures proper error handling and validates that your adapter fails gracefully when expected.

```javascript
// Example: Testing successful configuration
it('should configure and start adapter with valid configuration', function () {
    return new Promise(async (resolve, reject) => {
        // ... successful configuration test as shown above
    });
}).timeout(40000);

// Example: Testing failure scenarios
it('should NOT create daily states when daily is disabled', function () {
    return new Promise(async (resolve, reject) => {
        try {
            harness = getHarness();
            
            console.log('🔍 Step 1: Fetching adapter object...');
            const obj = await new Promise((res, rej) => {
                harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                    if (err) return rej(err);
                    res(o);
                });
            });
            
            // Configure to disable daily forecasts
            Object.assign(obj.native, {
                position: TEST_COORDINATES,
                createDaily: false  // ❌ Disabled
            });

            harness.objects.setObject(obj._id, obj);
            
            await harness.startAdapterAndWait();
            await wait(15000);
            
            const dailyStates = await harness.dbConnection.getStateIDs('your-adapter.0.summary.daily.*');
            
            if (dailyStates.length === 0) {
                console.log('✅ Test passed: No daily states created when disabled');
                resolve(true);
            } else {
                console.log(`❌ Test failed: ${dailyStates.length} daily states created when disabled`);
                reject(new Error('Daily states should not exist when createDaily is false'));
            }
            
        } catch (error) {
            reject(error);
        }
    });
}).timeout(40000);
```

### Testing with Mock Data

For adapters that connect to external APIs or hardware:

```javascript
// Create mock data files in test/mockData/ folder
const mockRoombaMissionData = require('./mockData/roomba-mission-complete.json');
const mockRoombaStatus = require('./mockData/roomba-status-cleaning.json');

// Use in tests to simulate API responses
it('should process mission data correctly', () => {
    const processedData = adapter.processMissionData(mockRoombaMissionData);
    expect(processedData.runtime).toBeDefined();
    expect(processedData.sqmCleaned).toBeGreaterThan(0);
});
```

## Development Patterns

### ioBroker Adapter Structure
- **Main adapter file**: `main.js` or `[adaptername].js` - Contains adapter logic and lifecycle
- **Admin interface**: `admin/` - Configuration UI and translations  
- **Library files**: `lib/` - Helper functions and modules
- **Package files**: `io-package.json` (adapter metadata) and `package.json` (npm metadata)

### Roomba Adapter Specific Patterns

#### Connection Management
```javascript
// Establish connection to Roomba with proper error handling
robot = new dorita980.Local(config.blid, config.robotpwd, config.ip);
robot.on('state', (state) => {
    // Process robot state updates
    this.processRobotState(state);
});

robot.on('mission', (mission) => {
    // Process mission data and create map if supported
    this.processMissionData(mission);
});
```

#### State Management Patterns
```javascript
// Use appropriate state structures for Roomba data
this.setObjectNotExists('commands.start', {
    type: 'state',
    common: {
        name: 'Start cleaning',
        type: 'boolean',
        role: 'button',
        write: true,
        read: false
    }
});

// Set robot status with proper error handling
this.setState('status.battery', { val: batteryLevel, ack: true });
```

#### Map Drawing and Canvas Operations
```javascript
// Canvas initialization for map drawing
const { createCanvas } = require('canvas');
const canvas = createCanvas(mapSize.width, mapSize.height);
const ctx = canvas.getContext('2d');

// Process positioning data for map visualization
if (mission && mission.pose) {
    const position = this.translatePosition(mission.pose);
    this.drawRoombaPosition(ctx, position);
}
```

### State Management
- Use `setState()` with proper acknowledgment flags
- Create object structures with `setObjectNotExists()` before setting states
- Implement proper data type validation
- Use appropriate state roles (indicator, button, value, etc.)

### Error Handling
- Always wrap external API calls in try-catch blocks
- Log errors with appropriate levels (adapter.log.error, .warn, .info, .debug)
- Implement connection retry logic with exponential backoff
- Gracefully handle device offline scenarios

### Resource Management  
- Clean up connections, timers, and resources in `unload()` method
- Properly close robot connections and stop intervals
- Clear canvas contexts and release memory

## Configuration and Setup

### Admin Interface
- Use materialize design framework for consistency with ioBroker
- Implement proper form validation
- Provide clear setup instructions and tooltips
- Support both manual and automatic credential discovery

### Roomba-Specific Configuration
```javascript
// Example configuration structure
native: {
    ip: '',           // Roomba IP address
    blid: '',         // Robot username/BLID
    robotpwd: '',     // Robot password  
    enableMapping: true,   // Enable map generation
    refreshInterval: 10000 // Status refresh interval
}
```

### Automatic Credential Discovery
```javascript
// UDP discovery for Roomba credentials
const dgram = require('dgram');
const discovery = dgram.createSocket('udp4');

discovery.on('message', (msg, rinfo) => {
    try {
        const robotInfo = JSON.parse(msg.toString());
        // Process discovered robot information
        this.handleDiscoveredRoomba(robotInfo, rinfo.address);
    } catch (error) {
        this.log.debug('Discovery message parse error: ' + error);
    }
});
```

## Logging Best Practices
- Use structured logging with consistent formatting
- Log connection events and API responses at debug level
- Log state changes and important events at info level  
- Log errors with full stack traces and context
- Include robot identification in log messages

```javascript
this.log.debug(`Roomba ${this.config.ip}: Received state update`);
this.log.info(`Mission started: ${mission.id}`);
this.log.warn(`Low battery: ${batteryLevel}%`);
this.log.error(`Connection failed: ${error.message}`);
```

## Translation and Internationalization
- All user-facing text must be translatable using the `words.js` system
- Use proper translation keys in admin interface
- Support common languages: en, de, ru, pt, nl, fr, it, es, pl
- Test translations in admin interface

## Security Considerations
- Store sensitive configuration (passwords, tokens) properly encrypted
- Validate all user inputs in admin interface
- Use secure connection methods when available
- Implement proper authentication handling

### Roomba Authentication Security
```javascript
// Secure credential storage
const robotpwd = this.config.robotpwd;
if (robotpwd && robotpwd.length > 0) {
    // Use credentials securely - don't log them
    this.connectToRoomba();
} else {
    this.log.error('Robot password not configured');
}
```

## Performance Guidelines
- Implement efficient polling intervals based on device capabilities
- Cache frequently accessed data appropriately  
- Use async/await for better code readability and error handling
- Minimize memory usage, especially for map data processing
- Implement proper cleanup to prevent memory leaks

## Version Management
- Follow semantic versioning (MAJOR.MINOR.PATCH)
- Update both `package.json` and `io-package.json` versions consistently
- Document changes in README changelog
- Test thoroughly before releasing updates

## Dependencies
- Keep dependencies updated but test compatibility
- Use official ioBroker adapter-core framework
- Prefer well-maintained libraries with good documentation
- Consider dependency size impact on installation time

### Roomba Adapter Dependencies
- `@karlvr/dorita980`: Core Roomba API communication
- `canvas`: Map drawing functionality (may require system dependencies)
- `@iobroker/adapter-core`: Standard adapter framework
- Standard Node.js modules: `dgram`, `tls`, `http`, `fs`

---

*This template was generated from the centralized ioBroker Copilot Instructions repository.*
*For updates and contributions, visit: https://github.com/DrozmotiX/ioBroker-Copilot-Instructions*