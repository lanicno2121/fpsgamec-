```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine.UI;
using UnityEngine;

public class EnemyHealthBarUI : MonoBehaviour {

    [Tooltip("血条填充块")]
    public Image healthSlider;

    private Transform cameraTransform;

    void Start()
    {
        cameraTransform = Camera.main.transform;
        healthSlider.fillAmount = 1;
    }
    void Update()
    {
        Vector3 dir = cameraTransform.position - transform.position;
        transform.rotation = Quaternion.LookRotation(-dir);
    }

    public void UpdateHealthBar(float healthRation)
    {
        healthSlider.fillAmount = healthRation;
    }
}
```
