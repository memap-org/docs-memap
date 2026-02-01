# Error Codes Reference

## Overview

MeMap uses standardized error codes across all microservices for consistent error handling. Each service has a dedicated code range to avoid conflicts.

## Response Format

### Error Response Structure

```json
{
  "code": 2006,
  "message": "User not found",
  "result": null
}
```

| Field   | Type   | Description                       |
| ------- | ------ | --------------------------------- |
| code    | int    | Numeric error code                |
| message | string | Human-readable error message      |
| result  | object | null for errors, data for success |

---

## Error Code Ranges

| Range     | Service          | Description                      |
| --------- | ---------------- | -------------------------------- |
| 1001-1999 | Roadmap Service  | Roadmap, nodes, permissions      |
| 2001-2999 | Profile Service  | Users, profiles, authentication  |
| 3001-3999 | Learning Service | Courses, lessons, progress       |
| 4001-4999 | File Service     | File upload/download             |
| 5001-5999 | AI Service       | AI generation, chat              |
| 7001-7999 | Payment Service  | Credits, subscriptions, payments |
| 9999      | All Services     | Uncategorized/unknown errors     |

---

## Roadmap Service (1xxx)

| Code | Name                                     | Message                                    | HTTP Status |
| ---- | ---------------------------------------- | ------------------------------------------ | ----------- |
| 1001 | ROADMAP_NOT_EXISTED                      | Roadmap doesn't exist                      | 400         |
| 1002 | METHOD_ARGUMENT_NOT_VALID                | Argument not valid                         | 400         |
| 1003 | UNAUTHENTICATED                          | Unauthenticated                            | 401         |
| 1004 | CANNOT_ACCESS_RESOURCE                   | You cannot access this roadmap             | 403         |
| 1005 | CANNOT_SET_OWNER_ID_FOR_PERMISSION       | Cannot set owner id for roadmap permission | 400         |
| 1006 | USER_NOT_EXIST_WHEN_UPDATE_PERMISSION    | User id not exist when update permission   | 400         |
| 1007 | USER_ALREADY_EXIST_IN_ROADMAP_PERMISSION | User already exists in roadmap permission  | 400         |
| 1008 | NODE_NOT_EXISTED                         | Node not exist                             | 400         |
| 1009 | FORBIDDEN                                | You cannot do this action                  | 403         |
| 1010 | ROADMAP_CATEGORY_NOT_EXIST               | Roadmap category not exist                 | 400         |
| 1011 | FILE_NOT_FOUND                           | File not found                             | 400         |

---

## Profile Service (2xxx)

| Code | Name                         | Message                      | HTTP Status |
| ---- | ---------------------------- | ---------------------------- | ----------- |
| 2001 | INVALID_CREDENTIALS          | Wrong username or password   | 400         |
| 2002 | METHOD_ARGUMENT_NOT_VALID    | Validation error             | 400         |
| 2003 | TOKEN_EXPIRED                | Token has been expired       | 401         |
| 2004 | UNAUTHENTICATED              | Unauthorized                 | 401         |
| 2005 | LOGOUT_FAIL                  | Logout failed                | 500         |
| 2006 | USER_NOT_FOUND               | User not found               | 400         |
| 2007 | EMAIL_EXISTED                | Email already exists         | 400         |
| 2008 | USERNAME_EXISTED             | Username already exists      | 400         |
| 2009 | USERNAME_IS_MISSING          | Please enter username        | 400         |
| 2010 | ACCESS_DENIED                | You do not have access       | 403         |
| 2011 | OLD_PASSWORD_INCORRECT       | Old password incorrect       | 400         |
| 2012 | USER_NOT_FOUND_WITH_EMAIL    | User not found with email    | 400         |
| 2013 | INVALID_USERNAME_OR_PASSWORD | Invalid username or password | 400         |
| 2014 | MESSAGE_LENGTH_INVALID       | Message length invalid       | 400         |
| 2015 | UPDATE_AVATAR_FAILED         | Update avatar failed         | 400         |
| 2016 | SEND_EMAIL_FAILED            | Send email failed            | 400         |

---

## Learning Service (3xxx)

| Code | Name                      | Message                    | HTTP Status |
| ---- | ------------------------- | -------------------------- | ----------- |
| 3001 | UNAUTHENTICATED           | Unauthenticated            | 401         |
| 3002 | ACCESS_DENIED             | Access denied              | 403         |
| 3003 | METHOD_ARGUMENT_NOT_VALID | Argument not valid         | 400         |
| 3004 | INVALID_REQUEST           | Invalid request            | 400         |
| 3101 | COURSE_NOT_FOUND          | Course not found           | 404         |
| 3102 | COURSE_ALREADY_EXISTS     | Course already exists      | 400         |
| 3103 | COURSE_NOT_AVAILABLE      | Course is not available    | 400         |
| 3201 | LESSON_NOT_FOUND          | Lesson not found           | 404         |
| 3202 | LESSON_ALREADY_EXISTS     | Lesson already exists      | 400         |
| 3203 | LESSON_NOT_ACCESSIBLE     | Lesson is not accessible   | 403         |
| 3301 | ENROLLMENT_NOT_FOUND      | Enrollment not found       | 404         |
| 3302 | ALREADY_ENROLLED          | Already enrolled in course | 400         |
| 3303 | NOT_ENROLLED              | Not enrolled in course     | 400         |
| 3401 | PROGRESS_NOT_FOUND        | Progress record not found  | 404         |
| 3402 | INVALID_PROGRESS_VALUE    | Invalid progress value     | 400         |
| 3501 | QUIZ_NOT_FOUND            | Quiz not found             | 404         |
| 3502 | QUIZ_ALREADY_SUBMITTED    | Quiz already submitted     | 400         |
| 3503 | QUIZ_TIME_EXPIRED         | Quiz time expired          | 400         |
| 3601 | CERTIFICATE_NOT_FOUND     | Certificate not found      | 404         |
| 3602 | CERTIFICATE_NOT_EARNED    | Certificate not earned yet | 400         |

---

## Payment Service (7xxx)

| Code | Name                      | Message                               | HTTP Status |
| ---- | ------------------------- | ------------------------------------- | ----------- |
| 7001 | AUTHENTICATION_REQUIRED   | Authentication required               | 401         |
| 7002 | ACCESS_DENIED             | Access denied                         | 403         |
| 7010 | INSUFFICIENT_CREDITS      | Insufficient credits                  | 402         |
| 7011 | INVALID_PACKAGE_ID        | Invalid package ID                    | 400         |
| 7012 | INVALID_PLAN_ID           | Invalid plan ID                       | 400         |
| 7013 | CREDIT_ACCOUNT_NOT_FOUND  | Credit account not found              | 404         |
| 7020 | PAYMENT_FAILED            | Payment failed                        | 400         |
| 7021 | INVALID_PAYMENT_METHOD    | Invalid payment method                | 400         |
| 7022 | CARD_DECLINED             | Card declined                         | 400         |
| 7023 | PAYMENT_CANCELLED         | Payment cancelled                     | 400         |
| 7024 | REFUND_FAILED             | Refund failed                         | 400         |
| 7030 | SUBSCRIPTION_NOT_FOUND    | Subscription not found                | 404         |
| 7031 | ALREADY_SUBSCRIBED        | Already subscribed to this plan       | 400         |
| 7032 | CANNOT_DOWNGRADE_TRIAL    | Cannot downgrade during trial period  | 400         |
| 7033 | SUBSCRIPTION_CANCELLED    | Subscription has been cancelled       | 400         |
| 7034 | SUBSCRIPTION_PAST_DUE     | Subscription payment is past due      | 400         |
| 7040 | INVALID_WEBHOOK_SIGNATURE | Invalid Stripe webhook signature      | 400         |
| 7041 | WEBHOOK_PROCESSING_FAILED | Failed to process webhook event       | 500         |
| 7050 | STORAGE_LIMIT_EXCEEDED    | Storage limit exceeded                | 400         |
| 7051 | AI_LIMIT_EXCEEDED         | AI generation limit exceeded          | 400         |
| 7052 | FEATURE_NOT_AVAILABLE     | Feature not available on current plan | 403         |
| 7060 | STRIPE_CUSTOMER_NOT_FOUND | Stripe customer not found             | 404         |
| 7061 | STRIPE_API_ERROR          | Stripe API error                      | 500         |
| 7099 | PAYMENT_SERVICE_ERROR     | Payment service error                 | 500         |

---

## General Errors (9xxx)

| Code | Name                    | Message             | HTTP Status |
| ---- | ----------------------- | ------------------- | ----------- |
| 9999 | UNCATEGORIZED_EXCEPTION | Uncategorized error | 500         |

---

## HTTP Status Code Mapping

| HTTP Status               | Usage                           |
| ------------------------- | ------------------------------- |
| 200 OK                    | Successful request              |
| 201 Created               | Resource created                |
| 400 Bad Request           | Validation error, invalid input |
| 401 Unauthorized          | Authentication required         |
| 403 Forbidden             | Insufficient permissions        |
| 404 Not Found             | Resource not found              |
| 409 Conflict              | Resource conflict (duplicate)   |
| 500 Internal Server Error | Unexpected server error         |

---

## Client-Side Error Handling

### JavaScript/TypeScript Example

```typescript
interface ApiResponse<T> {
  code: number;
  message: string;
  result: T | null;
}

async function handleApiCall<T>(response: Response): Promise<T> {
  const data: ApiResponse<T> = await response.json();

  if (data.code !== 0) {
    switch (data.code) {
      case 2003: // TOKEN_EXPIRED
      case 2004: // UNAUTHENTICATED
        // Redirect to login
        window.location.href = '/login';
        break;
      case 2006: // USER_NOT_FOUND
        throw new Error('User not found');
      case 2007: // EMAIL_EXISTED
        throw new Error('Email already registered');
      default:
        throw new Error(data.message || 'Unknown error');
    }
  }

  return data.result as T;
}
```

### Error Display Component

```typescript
const errorMessages: Record<number, string> = {
  1001: 'The roadmap you are looking for does not exist.',
  1004: 'You do not have permission to access this roadmap.',
  2006: 'User account not found.',
  2007: 'This email is already registered. Try logging in.',
  2011: 'The current password you entered is incorrect.',
  9999: 'An unexpected error occurred. Please try again.',
};

function getErrorMessage(code: number): string {
  return errorMessages[code] || 'Something went wrong.';
}
```

---

## Server-Side Error Handling

### Throwing Errors

```java
// Service layer
public User findUser(String userId) {
    return userRepository.findByUserId(userId)
        .orElseThrow(() -> new AppException(ErrorCode.USER_NOT_FOUND));
}

// With custom message
public void validateEmail(String email) {
    if (userRepository.existsByEmail(email)) {
        throw new AppException(ErrorCode.EMAIL_EXISTED);
    }
}
```

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(AppException.class)
    public ResponseEntity<ApiResponse<?>> handleAppException(AppException ex) {
        ErrorCode errorCode = ex.getErrorCode();
        return ResponseEntity
            .status(errorCode.getStatusCode())
            .body(ApiResponse.builder()
                .code(errorCode.getCode())
                .message(errorCode.getMessage())
                .build());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<?>> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(FieldError::getDefaultMessage)
            .collect(Collectors.joining(", "));

        return ResponseEntity
            .badRequest()
            .body(ApiResponse.builder()
                .code(ErrorCode.METHOD_ARGUMENT_NOT_VALID.getCode())
                .message(message)
                .build());
    }
}
```

---

## Adding New Error Codes

When adding new error codes:

1. **Choose the correct range** based on the service
2. **Use descriptive names** following the pattern: `ENTITY_ACTION_REASON`
3. **Write clear messages** that help users understand the issue
4. **Map to appropriate HTTP status**
5. **Update this documentation**

### Example: Adding a new error

```java
// In ErrorCode.java
public enum ErrorCode {
    // ... existing codes

    // New payment-related errors (2020-2029)
    PAYMENT_NOT_FOUND(2020, "Payment not found", HttpStatus.NOT_FOUND),
    PAYMENT_FAILED(2021, "Payment processing failed", HttpStatus.BAD_REQUEST),
    INSUFFICIENT_BALANCE(2022, "Insufficient balance", HttpStatus.BAD_REQUEST),
    ;
}
```
