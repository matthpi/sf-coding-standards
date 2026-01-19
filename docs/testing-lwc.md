# LWC Testing with Jest

For general testing principles and best practices that apply to both Apex and LWC, see [Testing](testing.md).

## Overview

Lightning Web Components use **Jest** as the testing framework. Jest provides a fast, isolated testing environment for JavaScript/LWC components with built-in mocking, assertions, and coverage reporting.

## Setup and Configuration

### Prerequisites

```bash
# Install Salesforce CLI and LWC testing utilities
npm install --save-dev @salesforce/sfdx-lwc-jest
```

### Running Tests

```bash
# Run all LWC tests
npm run test:unit

# Run tests in watch mode (re-runs on file changes)
npm run test:unit:watch

# Run tests with coverage report
npm run test:unit:coverage

# Run specific test file
npm run test:unit -- myComponent
```

### Test File Location

Test files must be in a `__tests__` directory within the component folder:

```
force-app/main/default/lwc/
├── myComponent/
│   ├── myComponent.html
│   ├── myComponent.js
│   ├── myComponent.js-meta.xml
│   └── __tests__/
│       └── myComponent.test.js
```

## Basic Test Structure

### Given/When/Then Pattern

```javascript
import { createElement } from 'lwc';
import MyComponent from 'c/myComponent';

describe('c-my-component', () => {
    afterEach(() => {
        // Clean up DOM after each test
        while (document.body.firstChild) {
            document.body.removeChild(document.body.firstChild);
        }
    });

    it('should display account name when data is provided', () => {
        // Given
        const element = createElement('c-my-component', {
            is: MyComponent
        });
        const mockAccount = { Id: '001xx000003DGb0', Name: 'Test Account' };

        // When
        document.body.appendChild(element);
        element.account = mockAccount;

        // Then
        return Promise.resolve().then(() => {
            const nameElement = element.shadowRoot.querySelector('.account-name');
            expect(nameElement.textContent).toBe('Test Account');
        });
    });
});
```

## Mocking Wire Services

### Mock @wire Decorated Properties

Use `@salesforce/sfdx-lwc-jest` utilities to mock wire adapters:

```javascript
import { createElement } from 'lwc';
import MyComponent from 'c/myComponent';
import { registerLdsTestWireAdapter } from '@salesforce/sfdx-lwc-jest';
import { getRecord } from 'lightning/uiRecordApi';

// Register wire adapter mock
const mockGetRecord = registerLdsTestWireAdapter(getRecord);

describe('c-my-component wire service', () => {
    afterEach(() => {
        while (document.body.firstChild) {
            document.body.removeChild(document.body.firstChild);
        }
    });

    it('should display record data when wire returns data', () => {
        // Given
        const mockRecord = {
            id: '001xx000003DGb0',
            fields: {
                Name: { value: 'Test Account' },
                Industry: { value: 'Technology' }
            }
        };

        // When
        const element = createElement('c-my-component', {
            is: MyComponent
        });
        element.recordId = '001xx000003DGb0';
        document.body.appendChild(element);

        // Emit data from wire
        mockGetRecord.emit(mockRecord);

        // Then
        return Promise.resolve().then(() => {
            const nameElement = element.shadowRoot.querySelector('.name');
            expect(nameElement.textContent).toBe('Test Account');
        });
    });

    it('should display error when wire returns error', () => {
        // Given
        const mockError = { body: { message: 'Record not found' } };

        // When
        const element = createElement('c-my-component', {
            is: MyComponent
        });
        document.body.appendChild(element);

        // Emit error from wire
        mockGetRecord.error(mockError);

        // Then
        return Promise.resolve().then(() => {
            const errorElement = element.shadowRoot.querySelector('.error');
            expect(errorElement).not.toBeNull();
            expect(errorElement.textContent).toContain('Record not found');
        });
    });
});
```

## Mocking Imperative Apex

Mock `@AuraEnabled` methods using Jest mocks:

```javascript
import { createElement } from 'lwc';
import MyComponent from 'c/myComponent';
import getAccounts from '@salesforce/apex/AccountController.getAccounts';

// Mock the Apex method
jest.mock(
    '@salesforce/apex/AccountController.getAccounts',
    () => {
        return {
            default: jest.fn()
        };
    },
    { virtual: true }
);

describe('c-my-component imperative apex', () => {
    afterEach(() => {
        while (document.body.firstChild) {
            document.body.removeChild(document.body.firstChild);
        }
        jest.clearAllMocks();
    });

    it('should load accounts when button is clicked', () => {
        // Given
        const mockAccounts = [
            { Id: '001xx000003DGb0', Name: 'Account 1' },
            { Id: '001xx000003DGb1', Name: 'Account 2' }
        ];
        getAccounts.mockResolvedValue(mockAccounts);

        const element = createElement('c-my-component', {
            is: MyComponent
        });
        document.body.appendChild(element);

        // When
        const button = element.shadowRoot.querySelector('lightning-button');
        button.click();

        // Then
        return Promise.resolve().then(() => {
            expect(getAccounts).toHaveBeenCalledTimes(1);
            const accountElements = element.shadowRoot.querySelectorAll('.account-item');
            expect(accountElements.length).toBe(2);
        });
    });

    it('should display error when apex call fails', () => {
        // Given
        const mockError = new Error('Server error');
        getAccounts.mockRejectedValue(mockError);

        const element = createElement('c-my-component', {
            is: MyComponent
        });
        document.body.appendChild(element);

        // When
        const button = element.shadowRoot.querySelector('lightning-button');
        button.click();

        // Then
        return Promise.resolve().then(() => {
            const errorElement = element.shadowRoot.querySelector('.error');
            expect(errorElement).not.toBeNull();
        });
    });
});
```

## Testing User Interactions

### Button Clicks and Events

```javascript
it('should fire custom event when button is clicked', () => {
    // Given
    const element = createElement('c-my-component', {
        is: MyComponent
    });
    document.body.appendChild(element);

    const handler = jest.fn();
    element.addEventListener('accountselected', handler);

    // When
    const button = element.shadowRoot.querySelector('lightning-button');
    button.click();

    // Then
    return Promise.resolve().then(() => {
        expect(handler).toHaveBeenCalledTimes(1);
        expect(handler.mock.calls[0][0].detail).toEqual({ accountId: '001xx000003DGb0' });
    });
});
```

### Input Field Changes

```javascript
it('should update property when input value changes', () => {
    // Given
    const element = createElement('c-my-component', {
        is: MyComponent
    });
    document.body.appendChild(element);

    // When
    const input = element.shadowRoot.querySelector('lightning-input');
    input.value = 'Test Value';
    input.dispatchEvent(new CustomEvent('change'));

    // Then
    return Promise.resolve().then(() => {
        expect(element.inputValue).toBe('Test Value');
    });
});
```

## Testing Conditional Rendering

```javascript
it('should show content when isLoading is false', () => {
    // Given
    const element = createElement('c-my-component', {
        is: MyComponent
    });

    // When
    document.body.appendChild(element);
    element.isLoading = false;

    // Then
    return Promise.resolve().then(() => {
        const content = element.shadowRoot.querySelector('.content');
        const spinner = element.shadowRoot.querySelector('lightning-spinner');

        expect(content).not.toBeNull();
        expect(spinner).toBeNull();
    });
});

it('should show spinner when isLoading is true', () => {
    // Given
    const element = createElement('c-my-component', {
        is: MyComponent
    });

    // When
    document.body.appendChild(element);
    element.isLoading = true;

    // Then
    return Promise.resolve().then(() => {
        const content = element.shadowRoot.querySelector('.content');
        const spinner = element.shadowRoot.querySelector('lightning-spinner');

        expect(content).toBeNull();
        expect(spinner).not.toBeNull();
    });
});
```

## Testing Getters and Computed Properties

```javascript
it('should compute full name from first and last name', () => {
    // Given
    const element = createElement('c-my-component', {
        is: MyComponent
    });
    document.body.appendChild(element);

    // When
    element.firstName = 'John';
    element.lastName = 'Doe';

    // Then
    return Promise.resolve().then(() => {
        const nameElement = element.shadowRoot.querySelector('.full-name');
        expect(nameElement.textContent).toBe('John Doe');
    });
});
```

## Best Practices

### Always Clean Up DOM

```javascript
afterEach(() => {
    // Remove all elements from DOM after each test
    while (document.body.firstChild) {
        document.body.removeChild(document.body.firstChild);
    }
});
```

### Use Promise.resolve() for Async Testing

LWC rendering is asynchronous. Always wait for promises to resolve:

```javascript
it('should update DOM after property change', () => {
    // Given/When
    const element = createElement('c-my-component', { is: MyComponent });
    document.body.appendChild(element);
    element.data = 'New Value';

    // Then - wait for rendering
    return Promise.resolve().then(() => {
        const textElement = element.shadowRoot.querySelector('.text');
        expect(textElement.textContent).toBe('New Value');
    });
});
```

### Mock All External Dependencies

- Mock `@wire` services using `registerLdsTestWireAdapter`
- Mock Apex methods using `jest.mock()`
- Mock platform events
- Mock custom labels and resources

### Test Accessibility

```javascript
it('should have proper aria labels', () => {
    const element = createElement('c-my-component', {
        is: MyComponent
    });
    document.body.appendChild(element);

    return Promise.resolve().then(() => {
        const button = element.shadowRoot.querySelector('lightning-button');
        expect(button.ariaLabel).toBe('Save Account');
    });
});
```

### Organize Tests by Functionality

```javascript
describe('c-my-component', () => {
    describe('Data Loading', () => {
        it('should load data on init', () => { });
        it('should handle load errors', () => { });
    });

    describe('User Interactions', () => {
        it('should handle button click', () => { });
        it('should validate input', () => { });
    });

    describe('Wire Services', () => {
        it('should display wire data', () => { });
        it('should handle wire errors', () => { });
    });
});
```

## Common Testing Patterns

### Test Data Factory

Create reusable mock data:

```javascript
// testUtils.js
export const createMockAccount = (overrides = {}) => {
    return {
        Id: '001xx000003DGb0',
        Name: 'Test Account',
        Industry: 'Technology',
        ...overrides
    };
};

export const createMockAccounts = (count) => {
    return Array.from({ length: count }, (_, i) =>
        createMockAccount({
            Id: `001xx000003DGb${i}`,
            Name: `Account ${i + 1}`
        })
    );
};

// myComponent.test.js
import { createMockAccount, createMockAccounts } from 'c/testUtils';

it('should display account', () => {
    const mockAccount = createMockAccount({ Name: 'Custom Name' });
    // Use mockAccount in test
});
```

### Flushable Promises

For multiple async operations:

```javascript
const flushPromises = () => new Promise(resolve => setImmediate(resolve));

it('should handle multiple async operations', async () => {
    const element = createElement('c-my-component', { is: MyComponent });
    document.body.appendChild(element);

    element.loadData();
    await flushPromises();

    const data = element.shadowRoot.querySelector('.data');
    expect(data).not.toBeNull();
});
```

## Resources

- [LWC Testing Documentation](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc.testing)
- [Jest Documentation](https://jestjs.io/docs/getting-started)
- [Salesforce LWC Recipes](https://github.com/trailheadapps/lwc-recipes)
